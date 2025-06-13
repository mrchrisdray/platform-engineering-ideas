# Platform Engineering Front Door - Implementation Guide

## 🏗️ Technical Architecture & Implementation

### Overview
This guide provides a complete implementation strategy for a platform engineering front door process that automates project onboarding, team setup, and resource provisioning through a pluggable, configuration-driven architecture.

## 🔧 Core Implementation Components

### 1. GitHub Actions Orchestration Engine

#### Main Workflow Structure
```yaml
# .github/workflows/project-provisioning.yml
name: Project Provisioning Pipeline
on:
  repository_dispatch:
    types: [provision-project]
  workflow_dispatch:
    inputs:
      config_payload:
        description: 'Base64 encoded project configuration'
        required: true
        type: string

env:
  CONFIG_REPO: 'platform-engineering/project-configs'
  NOTIFICATION_WEBHOOK: ${{ secrets.TEAMS_WEBHOOK_URL }}

jobs:
  # Job 1: Validate and Parse Configuration
  validate-config:
    runs-on: ubuntu-latest
    outputs:
      project-config: ${{ steps.parse.outputs.config }}
      validation-result: ${{ steps.validate.outputs.result }}
    steps:
      - name: Checkout Configuration Repository
        uses: actions/checkout@v4
        with:
          repository: ${{ env.CONFIG_REPO }}
          token: ${{ secrets.PLATFORM_TOKEN }}

      - name: Parse Project Configuration
        id: parse
        run: |
          if [ "${{ github.event_name }}" = "repository_dispatch" ]; then
            echo "config=$(echo '${{ github.event.client_payload.config }}' | jq -c .)" >> $GITHUB_OUTPUT
          else
            echo "config=$(echo '${{ github.event.inputs.config_payload }}' | base64 -d | jq -c .)" >> $GITHUB_OUTPUT
          fi

      - name: Validate Configuration Schema
        id: validate
        run: |
          echo "Validating project configuration..."
          python scripts/validate_config.py --config='${{ steps.parse.outputs.config }}'
          echo "result=success" >> $GITHUB_OUTPUT

  # Job 2: Check Approvals (if required)
  check-approvals:
    needs: validate-config
    runs-on: ubuntu-latest
    if: fromJson(needs.validate-config.outputs.project-config).approval_required == true
    steps:
      - name: Wait for Approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ secrets.GITHUB_TOKEN }}
          approvers: ${{ fromJson(needs.validate-config.outputs.project-config).approvers }}
          minimum-approvals: 1
          issue-title: "Project Approval Required: ${{ fromJson(needs.validate-config.outputs.project-config).metadata.name }}"

  # Job 3: GitHub Repository Setup
  setup-github:
    needs: [validate-config, check-approvals]
    if: always() && needs.validate-config.result == 'success' && (needs.check-approvals.result == 'success' || needs.check-approvals.result == 'skipped') && fromJson(needs.validate-config.outputs.project-config).components.github.enabled == true
    runs-on: ubuntu-latest
    outputs:
      repository-url: ${{ steps.create-repo.outputs.repository-url }}
    steps:
      - name: Create GitHub Repository
        id: create-repo
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          TEMPLATE=$(echo "$CONFIG" | jq -r '.components.github.template')
          DESCRIPTION=$(echo "$CONFIG" | jq -r '.metadata.description')
          
          # Create repository from template
          gh repo create "your-org/$PROJECT_NAME" \
            --template "your-org/$TEMPLATE" \
            --description "$DESCRIPTION" \
            --private
          
          echo "repository-url=https://github.com/your-org/$PROJECT_NAME" >> $GITHUB_OUTPUT

      - name: Configure Branch Protection
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          REQUIRED_REVIEWS=$(echo "$CONFIG" | jq -r '.components.github.required_reviewers // 2')
          
          if [ "$(echo "$CONFIG" | jq -r '.components.github.branch_protection')" = "true" ]; then
            gh api repos/your-org/$PROJECT_NAME/branches/main/protection \
              --method PUT \
              --field required_status_checks='{"strict":true,"contexts":[]}' \
              --field enforce_admins=true \
              --field required_pull_request_reviews="{\"required_approving_review_count\":$REQUIRED_REVIEWS,\"dismiss_stale_reviews\":true}" \
              --field restrictions=null
          fi

      - name: Setup Team Access
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          TEAM_NAME=$(echo "$CONFIG" | jq -r '.metadata.team')
          
          # Add team to repository
          gh api orgs/your-org/teams/$TEAM_NAME/repos/your-org/$PROJECT_NAME \
            --method PUT \
            --field permission=admin

  # Job 4: JFrog Artifactory Setup
  setup-artifactory:
    needs: [validate-config, check-approvals]
    if: always() && needs.validate-config.result == 'success' && (needs.check-approvals.result == 'success' || needs.check-approvals.result == 'skipped') && fromJson(needs.validate-config.outputs.project-config).components.artifactory.enabled == true
    runs-on: ubuntu-latest
    outputs:
      repositories: ${{ steps.create-repos.outputs.repositories }}
    steps:
      - name: Create Artifactory Repositories
        id: create-repos
        env:
          ARTIFACTORY_URL: ${{ secrets.ARTIFACTORY_URL }}
          ARTIFACTORY_TOKEN: ${{ secrets.ARTIFACTORY_TOKEN }}
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          REPOSITORIES=$(echo "$CONFIG" | jq -r '.components.artifactory.repositories[]')
          CREATED_REPOS=""
          
          for repo_type in $REPOSITORIES; do
            repo_name="${PROJECT_NAME}-${repo_type}"
            
            # Create repository
            curl -X PUT \
              "${ARTIFACTORY_URL}/artifactory/api/repositories/${repo_name}" \
              -H "Authorization: Bearer ${ARTIFACTORY_TOKEN}" \
              -H "Content-Type: application/json" \
              -d "{
                \"rclass\": \"local\",
                \"packageType\": \"${repo_type}\",
                \"description\": \"${repo_type} repository for ${PROJECT_NAME}\"
              }"
            
            CREATED_REPOS="${CREATED_REPOS}${repo_name},"
          done
          
          echo "repositories=${CREATED_REPOS%,}" >> $GITHUB_OUTPUT

      - name: Configure Access Permissions
        env:
          ARTIFACTORY_URL: ${{ secrets.ARTIFACTORY_URL }}
          ARTIFACTORY_TOKEN: ${{ secrets.ARTIFACTORY_TOKEN }}
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          TEAM_NAME=$(echo "$CONFIG" | jq -r '.metadata.team')
          
          # Create permission target
          curl -X PUT \
            "${ARTIFACTORY_URL}/artifactory/api/v2/security/permissions/${PROJECT_NAME}-permissions" \
            -H "Authorization: Bearer ${ARTIFACTORY_TOKEN}" \
            -H "Content-Type: application/json" \
            -d "{
              \"name\": \"${PROJECT_NAME}-permissions\",
              \"repo\": {
                \"include-patterns\": [\"**\"],
                \"exclude-patterns\": [],
                \"repositories\": [\"${PROJECT_NAME}-*\"],
                \"actions\": {
                  \"groups\": {
                    \"${TEAM_NAME}\": [\"read\", \"write\", \"annotate\", \"delete\"]
                  }
                }
              }
            }"

  # Job 5: Project Management Setup
  setup-project-management:
    needs: [validate-config, check-approvals]
    if: always() && needs.validate-config.result == 'success' && (needs.check-approvals.result == 'success' || needs.check-approvals.result == 'skipped') && fromJson(needs.validate-config.outputs.project-config).components.project_management.enabled == true
    runs-on: ubuntu-latest
    outputs:
      project-url: ${{ steps.setup-board.outputs.project-url }}
    steps:
      - name: Setup Jira Project
        id: setup-board
        env:
          JIRA_URL: ${{ secrets.JIRA_URL }}
          JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          PROJECT_KEY=$(echo "$PROJECT_NAME" | tr '[:lower:]' '[:upper:]' | tr '-' '_')
          SPRINT_DURATION=$(echo "$CONFIG" | jq -r '.components.project_management.sprint_duration // 14')
          
          # Create Jira project
          curl -X POST \
            "${JIRA_URL}/rest/api/3/project" \
            -H "Authorization: Bearer ${JIRA_TOKEN}" \
            -H "Content-Type: application/json" \
            -d "{
              \"key\": \"${PROJECT_KEY}\",
              \"name\": \"${PROJECT_NAME}\",
              \"projectTypeKey\": \"software\",
              \"description\": \"$(echo "$CONFIG" | jq -r '.metadata.description')\",
              \"lead\": \"$(echo "$CONFIG" | jq -r '.metadata.team_lead')\"
            }"
          
          echo "project-url=${JIRA_URL}/browse/${PROJECT_KEY}" >> $GITHUB_OUTPUT

      - name: Configure Sprints
        env:
          JIRA_URL: ${{ secrets.JIRA_URL }}
          JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          SPRINT_DURATION=$(echo "$CONFIG" | jq -r '.components.project_management.sprint_duration // 14')
          
          # Get board ID
          BOARD_ID=$(curl -s -H "Authorization: Bearer ${JIRA_TOKEN}" \
            "${JIRA_URL}/rest/agile/1.0/board?projectKeyOrId=${PROJECT_KEY}" | \
            jq -r '.values[0].id')
          
          # Create initial sprint
          START_DATE=$(date -I)
          END_DATE=$(date -d "+${SPRINT_DURATION} days" -I)
          
          curl -X POST \
            "${JIRA_URL}/rest/agile/1.0/sprint" \
            -H "Authorization: Bearer ${JIRA_TOKEN}" \
            -H "Content-Type: application/json" \
            -d "{
              \"name\": \"${PROJECT_NAME} Sprint 1\",
              \"startDate\": \"${START_DATE}\",
              \"endDate\": \"${END_DATE}\",
              \"originBoardId\": ${BOARD_ID}
            }"

  # Job 6: Finalize Setup
  finalize-setup:
    needs: [validate-config, setup-github, setup-artifactory, setup-project-management]
    if: always() && needs.validate-config.result == 'success'
    runs-on: ubuntu-latest
    steps:
      - name: Generate Project Documentation
        env:
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
          GITHUB_REPO: ${{ needs.setup-github.outputs.repository-url }}
          ARTIFACTORY_REPOS: ${{ needs.setup-artifactory.outputs.repositories }}
          PROJECT_URL: ${{ needs.setup-project-management.outputs.project-url }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          
          # Create README with all project information
          cat > project-summary.md << EOF
          # ${PROJECT_NAME} - Project Setup Complete
          
          ## Project Information
          - **Name**: ${PROJECT_NAME}
          - **Team**: $(echo "$CONFIG" | jq -r '.metadata.team')
          - **Description**: $(echo "$CONFIG" | jq -r '.metadata.description')
          
          ## Provisioned Resources
          - **GitHub Repository**: ${GITHUB_REPO}
          - **Artifactory Repositories**: ${ARTIFACTORY_REPOS}
          - **Project Management**: ${PROJECT_URL}
          
          ## Next Steps
          1. Clone the repository: \`git clone ${GITHUB_REPO}.git\`
          2. Review the README.md in the repository
          3. Access your project board: ${PROJECT_URL}
          4. Configure your local development environment
          
          ## Support
          For questions or issues, contact the Platform Engineering team.
          EOF

      - name: Send Notifications
        env:
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
          WEBHOOK_URL: ${{ secrets.TEAMS_WEBHOOK_URL }}
          GITHUB_REPO: ${{ needs.setup-github.outputs.repository-url }}
          PROJECT_URL: ${{ needs.setup-project-management.outputs.project-url }}
        run: |
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          TEAM_NAME=$(echo "$CONFIG" | jq -r '.metadata.team')
          
          # Send Teams/Slack notification
          curl -X POST \
            "${WEBHOOK_URL}" \
            -H "Content-Type: application/json" \
            -d "{
              \"title\": \"Project Setup Complete: ${PROJECT_NAME}\",
              \"text\": \"Your project has been successfully provisioned!\",
              \"sections\": [{
                \"facts\": [
                  {\"name\": \"Project\", \"value\": \"${PROJECT_NAME}\"},
                  {\"name\": \"Team\", \"value\": \"${TEAM_NAME}\"},
                  {\"name\": \"Repository\", \"value\": \"${GITHUB_REPO}\"},
                  {\"name\": \"Project Board\", \"value\": \"${PROJECT_URL}\"}
                ]
              }]
            }"

      - name: Update Project Registry
        env:
          CONFIG: ${{ needs.validate-config.outputs.project-config }}
          REGISTRY_TOKEN: ${{ secrets.PLATFORM_TOKEN }}
        run: |
          # Update internal project registry/database
          PROJECT_NAME=$(echo "$CONFIG" | jq -r '.metadata.name')
          
          # This could be a database update, API call, or file update
          echo "Updating project registry for ${PROJECT_NAME}"
```

### 2. Configuration Management System

#### Project Configuration Schema
```yaml
# schemas/project-config-schema.yml
apiVersion: v1
kind: ProjectConfiguration
metadata:
  name: string          # Project name (required)
  team: string          # Team name (required)
  description: string   # Project description
  business_unit: string # Business unit
  cost_center: string   # Cost center for billing
  team_lead: string     # Team lead email
  environment: array    # Target environments [dev, staging, prod]

approval_required: boolean  # Whether approval is needed
approvers: array           # List of approver emails

components:
  github:
    enabled: boolean
    template: string         # Template repository name
    branch_protection: boolean
    required_reviewers: integer
    teams: object           # team_name: permission_level
    
  artifactory:
    enabled: boolean
    repositories: array     # [docker, maven, npm, etc.]
    retention_policy: string # standard, extended, minimal
    xray_scanning: boolean
    
  project_management:
    enabled: boolean
    tool: string           # jira, azure-devops, github-projects
    sprint_duration: integer # Sprint duration in days
    board_type: string     # scrum, kanban
    project_key: string    # Custom project key
    
  secrets_management:
    enabled: boolean
    tool: string           # vault, azure-keyvault
    path: string           # Secret path
    
  monitoring:
    enabled: boolean
    dashboards: array      # Dashboard templates
    alerts: array          # Alert templates
    
  ci_cd:
    enabled: boolean
    pipeline_template: string
    environments: array
    
notifications:
  teams_webhook: string
  email_recipients: array
  slack_channel: string
```

#### Template System
```yaml
# templates/microservice-java.yml
apiVersion: v1
kind: ProjectTemplate
metadata:
  name: microservice-java
  description: Java Spring Boot microservice template
  version: 1.0.0

spec:
  default_configuration:
    components:
      github:
        enabled: true
        template: spring-boot-starter
        branch_protection: true
        required_reviewers: 2
      
      artifactory:
        enabled: true
        repositories: [docker, maven]
        retention_policy: standard
        xray_scanning: true
      
      project_management:
        enabled: true
        tool: jira
        sprint_duration: 14
        board_type: scrum
      
      ci_cd:
        enabled: true
        pipeline_template: java-microservice-pipeline
        environments: [dev, staging, prod]

  required_fields:
    - metadata.name
    - metadata.team
    - metadata.description

  validation_rules:
    - name: team_exists
      description: Team must exist in organization
      rule: metadata.team in allowed_teams
    
    - name: naming_convention
      description: Project name must follow naming convention
      rule: metadata.name matches "^[a-z][a-z0-9-]*[a-z0-9]$"
```

### 3. Developer Portal Integration

#### Frontend Configuration Form
```html
<!-- Developer Portal Project Request Form -->
<template>
  <div class="project-request-form">
    <h2>New Project Request</h2>
    
    <form @submit.prevent="submitRequest">
      <!-- Project Metadata -->
      <div class="form-section">
        <h3>Project Information</h3>
        <div class="form-group">
          <label for="project-name">Project Name *</label>
          <input 
            id="project-name" 
            v-model="config.metadata.name" 
            type="text" 
            required 
            pattern="^[a-z][a-z0-9-]*[a-z0-9]$"
            @blur="validateProjectName"
          >
          <span class="validation-error" v-if="errors.projectName">
            {{ errors.projectName }}
          </span>
        </div>
        
        <div class="form-group">
          <label for="team">Team *</label>
          <select id="team" v-model="config.metadata.team" required>
            <option value="">Select Team</option>
            <option v-for="team in availableTeams" :key="team.name" :value="team.name">
              {{ team.displayName }}
            </option>
          </select>
        </div>
        
        <div class="form-group">
          <label for="description">Description *</label>
          <textarea 
            id="description" 
            v-model="config.metadata.description" 
            required 
            rows="3"
          ></textarea>
        </div>
        
        <div class="form-group">
          <label for="template">Project Template</label>
          <select id="template" v-model="selectedTemplate" @change="applyTemplate">
            <option value="">Custom Configuration</option>
            <option v-for="template in templates" :key="template.name" :value="template.name">
              {{ template.displayName }} - {{ template.description }}
            </option>
          </select>
        </div>
      </div>
      
      <!-- Component Configuration -->
      <div class="form-section">
        <h3>Components</h3>
        
        <!-- GitHub Configuration -->
        <div class="component-config">
          <div class="component-header">
            <input 
              type="checkbox" 
              id="github-enabled" 
              v-model="config.components.github.enabled"
            >
            <label for="github-enabled">GitHub Repository</label>
          </div>
          
          <div v-if="config.components.github.enabled" class="component-details">
            <div class="form-group">
              <label for="github-template">Repository Template</label>
              <select id="github-template" v-model="config.components.github.template">
                <option value="">Choose template</option>
                <option value="microservice-java">Java Microservice</option>
                <option value="microservice-nodejs">Node.js Microservice</option>
                <option value="web-app-react">React Web Application</option>
                <option value="data-pipeline">Data Pipeline</option>
              </select>
            </div>
            
            <div class="form-group">
              <input 
                type="checkbox" 
                id="branch-protection" 
                v-model="config.components.github.branch_protection"
              >
              <label for="branch-protection">Enable Branch Protection</label>
            </div>
            
            <div class="form-group">
              <label for="required-reviewers">Required Reviewers</label>
              <input 
                id="required-reviewers" 
                type="number" 
                min="1" 
                max="5" 
                v-model="config.components.github.required_reviewers"
              >
            </div>
          </div>
        </div>
        
        <!-- Artifactory Configuration -->
        <div class="component-config">
          <div class="component-header">
            <input 
              type="checkbox" 
              id="artifactory-enabled" 
              v-model="config.components.artifactory.enabled"
            >
            <label for="artifactory-enabled">JFrog Artifactory</label>
          </div>
          
          <div v-if="config.components.artifactory.enabled" class="component-details">
            <div class="form-group">
              <label>Repository Types</label>
              <div class="checkbox-group">
                <label>
                  <input type="checkbox" value="docker" v-model="config.components.artifactory.repositories"> Docker
                </label>
                <label>
                  <input type="checkbox" value="maven" v-model="config.components.artifactory.repositories"> Maven
                </label>
                <label>
                  <input type="checkbox" value="npm" v-model="config.components.artifactory.repositories"> NPM
                </label>
                <label>
                  <input type="checkbox" value="pypi" v-model="config.components.artifactory.repositories"> PyPI
                </label>
              </div>
            </div>
            
            <div class="form-group">
              <label for="retention-policy">Retention Policy</label>
              <select id="retention-policy" v-model="config.components.artifactory.retention_policy">
                <option value="minimal">Minimal (30 days)</option>
                <option value="standard">Standard (90 days)</option>
                <option value="extended">Extended (365 days)</option>
              </select>
            </div>
          </div>
        </div>
        
        <!-- Project Management Configuration -->
        <div class="component-config">
          <div class="component-header">
            <input 
              type="checkbox" 
              id="project-mgmt-enabled" 
              v-model="config.components.project_management.enabled"
            >
            <label for="project-mgmt-enabled">Project Management</label>
          </div>
          
          <div v-if="config.components.project_management.enabled" class="component-details">
            <div class="form-group">
              <label for="pm-tool">Tool</label>
              <select id="pm-tool" v-model="config.components.project_management.tool">
                <option value="jira">Jira</option>
                <option value="azure-devops">Azure DevOps</option>
                <option value="github-projects">GitHub Projects</option>
              </select>
            </div>
            
            <div class="form-group">
              <label for="sprint-duration">Sprint Duration (days)</label>
              <input 
                id="sprint-duration" 
                type="number" 
                min="7" 
                max="30" 
                v-model="config.components.project_management.sprint_duration"
              >
            </div>
            
            <div class="form-group">
              <label for="board-type">Board Type</label>
              <select id="board-type" v-model="config.components.project_management.board_type">
                <option value="scrum">Scrum</option>
                <option value="kanban">Kanban</option>
              </select>
            </div>
          </div>
        </div>
      </div>
      
      <!-- Cost Estimation -->
      <div class="form-section">
        <h3>Cost Estimation</h3>
        <div class="cost-breakdown">
          <div class="cost-item">
            <span>GitHub Repository</span>
            <span>${{ calculateGitHubCost() }}/month</span>
          </div>
          <div class="cost-item">
            <span>Artifactory Storage</span>
            <span>${{ calculateArtifactoryCost() }}/month</span>
          </div>
          <div class="cost-item">
            <span>Project Management</span>
            <span>${{ calculateProjectMgmtCost() }}/month</span>
          </div>
          <div class="cost-total">
            <span>Total Estimated Cost</span>
            <span>${{ calculateTotalCost() }}/month</span>
          </div>
        </div>
      </div>
      
      <!-- Approval Information -->
      <div class="form-section" v-if="requiresApproval">
        <h3>Approval Required</h3>
        <p>This project requires approval from:</p>
        <ul>
          <li v-for="approver in getRequiredApprovers()" :key="approver">
            {{ approver }}
          </li>
        </ul>
      </div>
      
      <!-- Submit Button -->
      <div class="form-actions">
        <button type="button" @click="previewConfiguration">Preview Configuration</button>
        <button type="submit" :disabled="!isFormValid">
          {{ require
