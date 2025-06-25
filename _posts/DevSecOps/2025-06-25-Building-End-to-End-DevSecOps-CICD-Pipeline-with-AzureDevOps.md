---
title: "Building an End-to-End DevSecOps CI/CD Pipeline with Azure DevOps"
classes: wide
header:
  teaser: /assets/images/DevSecOps/CICD-Pipeline/pipeline-teaser.jpg
  # teaser: https://hackmd.io/_uploads/ByjqS_FExg.jpg
ribbon: gray
description: "A real-world guide to designing and automating a secure CI/CD pipeline in Azure DevOps, integrating SAST, DAST, SCA, secrets scanning, container security, and centralized vulnerability management with DefectDojo."
categories:
  - DevSecOps
toc: true
---
<!-- ![pipeline-teaser](https://hackmd.io/_uploads/ByjqS_FExg.jpg) -->


# 🚀 Building an End-to-End DevSecOps CI/CD Pipeline with Azure DevOps

> A comprehensive walkthrough of designing, building, and automating secure CI/CD workflows with static and dynamic security testing, secrets detection, dependency scanning, container security, and centralized vulnerability management using DefectDojo.

This guide details the creation of a production-grade **DevSecOps CI/CD pipeline** in **Azure DevOps**, developed through the collaboration between Michael and Hossam—hence the ***0xCoolSAM***. The pipeline automates security scanning across source code, dependencies, containers, and runtime environments, with all findings centralized in **DefectDojo** for unified vulnerability management.

![Pipeline Overview](/assets/images/DevSecOps/CICD-Pipeline/pipeline-overview.jpg)

<!-- ![Pipeline Overview1](https://hackmd.io/_uploads/SkXyOBtEex.jpg) -->
<!-- ![Pipeline Overview2](https://hackmd.io/_uploads/SkXfKrYExx.jpg) -->

---

## ✨ Overview

Modern software development demands security at every stage. Our pipeline integrates multiple security tools to enforce **Security by Design**, covering:

- **Secrets Scanning**: Detect hardcoded credentials using GitLeaks and TruffleHog.
- **Software Composition Analysis (SCA)**: Identify vulnerabilities in dependencies with Sonatype Nexus IQ.
- **Static Application Security Testing (SAST)**: Analyze code with Fortify ScanCentral SAST.
- **Container Security**: Scan Docker images using Trivy.
- **Dynamic Application Security Testing (DAST)**: Test runtime vulnerabilities with Fortify ScanCentral DAST.
- **Vulnerability Management**: Aggregate and track findings using DefectDojo.
- **Custom Automation**: A private Azure DevOps extension for seamless DefectDojo integration.

This post provides a step-by-step guide, complete with pipeline configurations and insights from our implementation.

---

## 🛠️ Stack Overview

| Stage                     | Tool                        | Purpose                                           |
|---------------------------|-----------------------------|--------------------------------------------------|
| Secrets Scanning          | GitLeaks, TruffleHog        | Detect hardcoded secrets and sensitive data.          |
| SCA (Dependency) Scanning | Sonatype Nexus IQ           | Identify open-source components, their versions, and associated security vulnerabilities, license risks, and quality issues.   |
| SAST                      | Fortify  ScanCentral SAST | Perform static code analysis to identify security vulnerabilities in source code.                     |
| Container Image Scanning  | Trivy                       | Finds and detects vulnerabilities, misconfigurations, and secrets in containers.             |
| DAST                      | Fortify ScanCentral DAST             | Analyzes a running application to identify security vulnerabilities by simulating real-world attacks. |
| Vulnerability Aggregation | DefectDojo                  | Centralize, track, and correlate findings         |

---

## 🧱 Pipeline Architecture

The pipeline, defined in an Azure DevOps YAML file, runs on a self-hosted agent (named "Contabo") and consists of the following stages:

1. **Secrets Detection**: GitLeaks and TruffleHog scan the source code, producing JSON reports.
2. **Build & SCA**: Maven builds the Java application, and Nexus IQ scans dependencies.
3. **SAST**: Fortify ScanCentral SAST analyzes the codebase, generating an `.fpr` file.
4. **Container Scanning**: Trivy scans Docker images for vulnerabilities.
5. **DAST**: The application is deployed via Docker Compose, scanned with Fortify SC DAST, and torn down post-scan.
6. **Vulnerability Aggregation**: A custom Azure DevOps extension uploads all scan results to DefectDojo.

![Azure DevOps Pipeline](/assets/images/DevSecOps/CICD-Pipeline/pipeline-view.jpg)
<!-- ![Azure DevOps Pipeline](https://hackmd.io/_uploads/BJ045StEgg.jpg) -->

---

## 🧩 Prerequisites

Before setting up the pipeline, ensure the following are in place:

- **Azure DevOps Account**: Configured with a project and repository.
- **Self-Hosted Agent**: Named "Contabo" with the following tools installed:
  - GitLeaks (`gitleaks`)
  - TruffleHog (`trufflehog`)
  - Maven (`mvn`)
  - Docker (`docker`)
  - Fortify CLI (`fcli`)
  - Trivy (`trivy`)
- **Dependencies**:
  - Sonatype Nexus IQ server.
  - Fortify SSC and ScanCentral servers.
  - DefectDojo instance (configured via environment variable `DEFECTDOJO_HOST`).

![Prerequisites](/assets/images/DevSecOps/CICD-Pipeline/prerequisites.jpg)
<!-- ![Prerequisites](https://hackmd.io/_uploads/SkgskiHKNgx.jpg) -->

---

## 📋 Pipeline Configuration

Below are the detailed configurations for each pipeline stage, with sensitive values replaced by environment variables.

### **1. Secrets Detection**

#### **GitLeaks**
Detects sensitive data in the source code.

```yaml
- job: gitleaksScan
  displayName: "GitLeaks Scan"
  continueOnError: true
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - script: |
        echo "🔎 Running GitLeaks..."
        mkdir -p $(Build.ArtifactStagingDirectory)/gitleaks
        gitleaks detect \
          --source "$(Build.SourcesDirectory)" \
          --report-format json \
          --report-path "$(Build.ArtifactStagingDirectory)/gitleaks/gitleaks-report.json" \
          || true
      displayName: 'Run GitLeaks (Ignore Warnings)'
    - task: PublishPipelineArtifact@1
      displayName: "Publish GitLeaks Scan Report"
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)/gitleaks'
        artifact: 'GitLeaksScanReport'
```
We named it `Run GitLeaks (Ignore Warnings)` because we used the locally installed GitLeaks on our Contabo agent to avoid the warnings that are triggered when secrets are detected using the GitLeaks extension in Azure DevOps.

<!-- ![GitLeaks Report](/assets/images/DevSecOps/CICD-Pipeline/gitleaks-report.png) -->
---
#### **TruffleHog**
Complements GitLeaks for additional secret detection.

```yaml
- job: trufflehogScan
  displayName: "TruffleHog Scan"
  continueOnError: true
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - checkout: self
    - script: |
        echo "🔍 Running TruffleHog..."
        mkdir -p $(Build.ArtifactStagingDirectory)/trufflehog
        trufflehog git file://$(Build.SourcesDirectory) --json --no-update \
          > $(Build.ArtifactStagingDirectory)/trufflehog/trufflehog-report.json
      displayName: "Secrets Detection Using TruffleHog Scan"
    - task: PublishPipelineArtifact@1
      displayName: "Publish TruffleHog Scan Report"
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)/trufflehog'
        artifact: 'TruffleHogScanReport'
```

<!-- ![TruffleHog Report](/assets/images/DevSecOps/CICD-Pipeline/trufflehog-report.png) -->

---

### **2. Build & SCA**

Builds the Java application using Maven and scans dependencies with Nexus IQ.

```yaml
- job: buildAndNexusScan
  displayName: "Maven Build & Nexus IQ SCA Scan"
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - checkout: self
    - task: Maven@4
      displayName: 'Maven Build'
      inputs:
        mavenPomFile: 'pom.xml'
        goals: 'clean package -DskipTests'
        publishJUnitResults: true
        testResultsFiles: '**/surefire-reports/TEST-*.xml'
    - task: NexusIqPipelineTask@2
      displayName: 'SCA Using Nexus IQ Scan'
      inputs:
        nexusIqService: 'Sonatype'
        scanTargets: '**/target/**/*.jar, **/target/**/*.war'
        organizationId: '$(NEXUS_IQ_ORG_ID)'
        applicationId: 'java'
        stage: 'Build'
        resultFile: 'javasec.json'
        ignoreSystemError: true
        enableDebugLogging: true
        acceptIqServerSelfSignedCertificates: true
    - script: |
        echo "Copying Nexus IQ result..."
        mkdir -p $(Build.ArtifactStagingDirectory)/nexus
        find $(Agent.WorkFolder)/_tasks/NexusIqPipelineTask*/ -name javasec.json -exec cp {} $(Build.ArtifactStagingDirectory)/nexus/javasec.json \;
      displayName: 'Copy Nexus IQ Result to Staging Directory'
    - script: |
        echo "🔍 Extracting reportDataUrl from javasec.json..."
        JSON_FILE="$(Build.ArtifactStagingDirectory)/nexus/javasec.json"
        RAW_URL=$(python3 -c "import json; f=open('${JSON_FILE}'); j=json.load(f); print(j['reportDataUrl'])")
        echo "📥 Downloading raw report from: $RAW_URL"
        curl -u "$(NEXUS_IQ_USERNAME):$(NEXUS_IQ_PASSWORD)" \
             -o "$(Build.ArtifactStagingDirectory)/nexus/raw-report.json" \
             "$RAW_URL"
      displayName: 'Download Nexus Raw Scan JSON with Authentication'
    - task: PublishPipelineArtifact@1
      displayName: "Publish All Nexus IQ Scan Artifacts"
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)/nexus'
        artifact: 'NexusFullScanReport'
```

![Nexus IQ Report](/assets/images/DevSecOps/CICD-Pipeline/nexus-iq-report1.png)
<br>
![Nexus IQ Report](/assets/images/DevSecOps/CICD-Pipeline/nexus-iq-report2.png)
<!-- ![Nexus IQ Report1](https://hackmd.io/_uploads/S1MFQdFVex.png)
![Nexus IQ Report2](https://hackmd.io/_uploads/H1OYmOKVll.png) -->

---

### **3. SAST with Fortify**

Performs static code analysis using Fortify ScanCentral SAST.

```yaml
- job: fortifySAST
  displayName: "SAST Using Fortify SC SAST Scan"
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - checkout: self
    - task: FortifyScanCentralSAST@7
      displayName: "Run Fortify SAST Scan"
      inputs:
        scanCentralCtrlUrl: '$(SCANCENTRAL_CTRL_URL)'
        scanCentralClientToken: '$(SCANCENTRAL_CLIENT_TOKEN)'
        sscUrl: '$(SSC_URL)'
        sscCiToken: '$(SSC_CI_TOKEN)'
        uploadToSSC: true
        applicationIdentifierType: 'byId'
        applicationVersionId: '$(FORTIFY_APP_VERSION_ID)'
        autoDetectBuildTool: true
        block: true
        outputFile: 'java_sec_sast.fpr'
    - script: |
        mkdir -p $(Build.ArtifactStagingDirectory)/SAST
        mv $(Build.SourcesDirectory)/java_sec_sast.fpr $(Build.ArtifactStagingDirectory)/SAST/
      displayName: 'Move Fortify FPR to Staging Directory'
    - task: PublishPipelineArtifact@1
      displayName: "Publish Fortify FPR Report"
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)/SAST/java_sec_sast.fpr'
        artifact: 'FortifyFPRReport'
```

![Fortify SAST Results](/assets/images/DevSecOps/CICD-Pipeline/fortify-sast-results1.png)
<br>
![Fortify SAST Results](/assets/images/DevSecOps/CICD-Pipeline/fortify-sast-results2.png)
<!-- ![Fortify SAST Results1](https://hackmd.io/_uploads/ByW3muFEle.png)
![Fortify SAST Results2](https://hackmd.io/_uploads/H1aT7OtEll.png) -->

---

### **4. Container Image Scanning with Trivy**

Scans Docker images for vulnerabilities.

```yaml
- job: trivyScan
  displayName: "Container Image Scanning Using Trivy Scan"
  continueOnError: true
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - checkout: self
    - script: |
        mkdir -p $(Build.ArtifactStagingDirectory)/Trivy
        trivy image $(DOCKER_IMAGE_NAME) -f json --output $(Build.ArtifactStagingDirectory)/Trivy/trivyreport.json
      displayName: 'Run Trivy Scan'
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)/Trivy'
        artifact: 'TrivyScanReport'
```

<!-- ![Trivy Report](/assets/images/DevSecOps/CICD-Pipeline/trivy-report.png) -->

---

### **5. DAST with Fortify**

Deploys the application using Docker Compose and performs dynamic scanning.

```yaml
- job: fortifyDAST
  displayName: "Start App for Fortify DAST"
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - checkout: self
    - task: DockerCompose@1
      displayName: 'Start App via Docker Compose'
      inputs:
        containerregistrytype: 'Container Registry'
        dockerComposeFile: 'docker-compose.yml'
        action: 'Run a Docker Compose command'
        dockerComposeCommand: 'up -d'
    - script: |
        echo "Waiting for services..."
        sleep 15
      displayName: 'Wait for Containers'
    - task: DockerCompose@1
      displayName: 'Check Running Containers'
      inputs:
        containerregistrytype: 'Container Registry'
        dockerComposeFile: 'docker-compose.yml'
        action: 'Run a Docker Compose command'
        dockerComposeCommand: 'ps'
```

```yaml
- job: fortifyDAST_Start
  displayName: "DAST Using Fortify SC DAST Scan"
  dependsOn: fortifyDAST
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - task: FortifyScanCentralDAST@7
      displayName: 'Trigger DAST Scan'
      inputs:
        scanCentralDastApiUrl: '$(SCANCENTRAL_DAST_API_URL)'
        scanCentralCiCdToken: '$(SCANCENTRAL_CICD_TOKEN)'
        sscCiToken: '$(SSC_CI_TOKEN)'
        overrides: '{"name": "Java-Sec"}'
```

```yaml
- job: fortifyDAST_Wait
  displayName: "Wait for Fortify SC DAST Completion"
  dependsOn: fortifyDAST_Start
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - script: |
        echo "🔐 Authenticating to SSC..."
        fcli ssc session login \
          --token "$(SSC_CI_TOKEN)" \
          --url "$(SSC_URL)" \
          --insecure
        echo "📄 Listing available scans..."
        scan_id=$(fcli sc-dast scan list --store=id --output json | jq 'sort_by(.id) | reverse | .[0].id')
        echo "✅ Latest Scan ID: $scan_id"
        echo "⏳ Waiting for scan to finish..."
        fcli sc-dast scan wait-for "$scan_id" --interval=30s
      displayName: 'Wait for DAST Scan Completion'
    - task: DockerCompose@1
      displayName: 'Tear Down Containers'
      inputs:
        containerregistrytype: 'Container Registry'
        dockerComposeFile: 'docker-compose.yml'
        action: 'Run a Docker Compose command'
        dockerComposeCommand: 'down'
```

![Fortify DAST Results](/assets/images/DevSecOps/CICD-Pipeline/fortify-dast-results1.png)
<br>
![Fortify DAST Results](/assets/images/DevSecOps/CICD-Pipeline/fortify-dast-results2.png)
<!-- ![Fortify DAST Results1](https://hackmd.io/_uploads/H1uy4utEeg.png)
![Fortify DAST Results2](https://hackmd.io/_uploads/BkSx4dYEex.png) -->

---

### **6. Vulnerability Aggregation in DefectDojo**

Uploads all scan results to DefectDojo using a custom Azure DevOps extension.

```yaml
- job: DefectDojo
  displayName: "Upload All Reports to DefectDojo"
  dependsOn:
    - gitleaksScan
    - trufflehogScan
    - trivyScan
    - buildAndNexusScan
    - fortifySAST
    - fortifyDAST_Wait
  pool:
    name: 'DevSecOps'
    demands:
      - agent.name -equals Contabo
  steps:
    - task: DownloadPipelineArtifact@2
      displayName: "Download GitLeaks Scan Report"
      inputs:
        artifact: 'GitLeaksScanReport'
        path: '$(Build.ArtifactStagingDirectory)/gitleaks'
    - task: UploadToDefectDojo@1
      displayName: "Upload GitLeaks Report"
      continueOnError: true
      inputs:
        host: '$(DEFECTDOJO_HOST)'
        api_key: '$(DEFECTDOJO_API_KEY)'
        engagement_id: '$(DEFECTDOJO_ENGAGEMENT_ID)'
        product_id: '$(DEFECTDOJO_PRODUCT_ID)'
        lead_id: '$(DEFECTDOJO_LEAD_ID)'
        environment: 'Development'
        result_file: '$(Build.ArtifactStagingDirectory)/gitleaks/gitleaks-report.json'
        scanner: 'Gitleaks Scan'
    - task: DownloadPipelineArtifact@2
      displayName: "Download TruffleHog Scan Report"
      inputs:
        artifact: 'TruffleHogScanReport'
        path: '$(Build.ArtifactStagingDirectory)/trufflehog'
    - task: UploadToDefectDojo@1
      displayName: "Upload TruffleHog Report"
      continueOnError: true
      inputs:
        host: '$(DEFECTDOJO_HOST)'
        api_key: '$(DEFECTDOJO_API_KEY)'
        engagement_id: '$(DEFECTDOJO_ENGAGEMENT_ID)'
        product_id: '$(DEFECTDOJO_PRODUCT_ID)'
        lead_id: '$(DEFECTDOJO_LEAD_ID)'
        environment: 'Development'
        result_file: '$(Build.ArtifactStagingDirectory)/trufflehog/trufflehog-report.json'
        scanner: 'Trufflehog Scan'
    - task: DownloadPipelineArtifact@2
      displayName: "Download Nexus Scan Report"
      inputs:
        artifact: 'NexusFullScanReport'
        path: '$(Build.ArtifactStagingDirectory)/nexus'
    - task: UploadToDefectDojo@1
      displayName: "Upload Nexus IQ SCA Report"
      continueOnError: true
      inputs:
        host: '$(DEFECTDOJO_HOST)'
        api_key: '$(DEFECTDOJO_API_KEY)'
        engagement_id: '$(DEFECTDOJO_ENGAGEMENT_ID)'
        product_id: '$(DEFECTDOJO_PRODUCT_ID)'
        lead_id: '$(DEFECTDOJO_LEAD_ID)'
        environment: 'Development'
        result_file: '$(Build.ArtifactStagingDirectory)/nexus/raw-report.json'
        scanner: 'Sonatype Application Scan'
    - task: DownloadPipelineArtifact@2
      displayName: "Download SAST Scan Report"
      inputs:
        artifact: 'FortifyFPRReport'
        path: '$(Build.ArtifactStagingDirectory)/SAST'
    - task: UploadToDefectDojo@1
      displayName: "Upload Fortify SAST FPR"
      continueOnError: true
      inputs:
        host: '$(DEFECTDOJO_HOST)'
        api_key: '$(DEFECTDOJO_API_KEY)'
        engagement_id: '$(DEFECTDOJO_ENGAGEMENT_ID)'
        product_id: '$(DEFECTDOJO_PRODUCT_ID)'
        lead_id: '$(DEFECTDOJO_LEAD_ID)'
        environment: 'Development'
        result_file: '$(Build.ArtifactStagingDirectory)/SAST/java_sec_sast.fpr'
        scanner: 'Fortify Scan'
    - task: DownloadPipelineArtifact@2
      displayName: "Download Trivy Scan Report"
      inputs:
        artifact: 'TrivyScanReport'
        path: '$(Build.ArtifactStagingDirectory)/Trivy'
    - task: UploadToDefectDojo@1
      displayName: "Upload Trivy Container Scan"
      continueOnError: true
      inputs:
        host: '$(DEFECTDOJO_HOST)'
        api_key: '$(DEFECTDOJO_API_KEY)'
        engagement_id: '$(DEFECTDOJO_ENGAGEMENT_ID)'
        product_id: '$(DEFECTDOJO_PRODUCT_ID)'
        lead_id: '$(DEFECTDOJO_LEAD_ID)'
        environment: 'Development'
        result_file: '$(Build.ArtifactStagingDirectory)/Trivy/trivyreport.json'
        scanner: 'Trivy Scan'
```

---

## 🧩 Custom Azure DevOps Extension

Initially, scan results were manually uploaded to DefectDojo, which was time-consuming. To address this, we developed a **custom Azure DevOps extension** to automate the upload of GitLeaks, TruffleHog, Nexus IQ, Fortify SAST, and Trivy reports. The extension:

- Fetches pipeline artifacts.
- Uses the DefectDojo API to post results.
- Tags each result with product and environment metadata (e.g., `Development`, `$(DEFECTDOJO_ENGAGEMENT_ID)`).

This extension is currently private but is being enhanced for potential public release.

<!-- ![Custom Extension](/assets/images/DevSecOps/CICD-Pipeline/custom-extension.png) -->

---

## 🔍 Verifying the Pipeline

### **Check Pipeline Status**
Monitor the pipeline execution in Azure DevOps:

```bash
az pipelines runs list --project $(AZURE_DEVOPS_PROJECT_NAME)
```

![Pipeline Status](/assets/images/DevSecOps/CICD-Pipeline/pipeline-status.png)
<!-- ![Pipeline Status](https://hackmd.io/_uploads/ryMTqHY4lg.png) -->


### **View DefectDojo Reports**
Access the DefectDojo dashboard to review consolidated vulnerabilities

![DefectDojo Dashboard](/assets/images/DevSecOps/CICD-Pipeline/defectdojo-dashboard1.png)
<br>
![DefectDojo Dashboard](/assets/images/DevSecOps/CICD-Pipeline/defectdojo-dashboard2.png)
<!-- ![DefectDojo Dashboard1](https://hackmd.io/_uploads/BkP-V_Y4lg.png)
![DefectDojo Dashboard2](https://hackmd.io/_uploads/rJgzEuF4lg.png) -->

---

## 🛠️ Troubleshooting

- **Secrets Scanning Failures**: Verify GitLeaks and TruffleHog are installed on the agent and paths are correct.
- **Nexus IQ Issues**: Check `reportDataUrl` extraction and ensure `$(NEXUS_IQ_USERNAME)` and `$(NEXUS_IQ_PASSWORD)` are set.
- **Fortify Errors**: Validate `$(SCANCENTRAL_CTRL_URL)`, `$(SSC_URL)`, `$(SCANCENTRAL_CLIENT_TOKEN)`, and `$(SSC_CI_TOKEN)`.
- **DefectDojo Uploads**: Confirm `$(DEFECTDOJO_HOST)`, `$(DEFECTDOJO_API_KEY)`, `$(DEFECTDOJO_ENGAGEMENT_ID)`, and `$(DEFECTDOJO_PRODUCT_ID)` are correctly configured.
- **Container Issues**: Ensure Docker Compose file accuracy and registry credentials are set in Azure DevOps secrets.

---

## 💭 Key Takeaways

This project demonstrates a fully automated DevSecOps pipeline enforcing **Security by Design**. Key lessons:

- **Automate Everything**: From scans to cleanups, automation reduces manual effort.
- **Integrate Early**: Security checks at the commit level catch issues sooner.
- **Centralize Results**: DefectDojo provides a single pane of glass for vulnerability management.
- **Continuous Improvement**: Each scan offers insights to harden the pipeline.

---

## 🔗 Useful References

- [GitLeaks](https://github.com/gitleaks/gitleaks)
- [TruffleHog](https://github.com/trufflesecurity/trufflehog)
- [Sonatype Nexus IQ](https://help.sonatype.com/en/sonatype-lifecycle.html)
- [Sonatype for Azure DevOps](https://marketplace.visualstudio.com/items?itemName=SonatypeIntegrations.nexus-iq-azure-extension)
- [Fortify SSC](https://www.microfocus.com/documentation/fortify-software-security-center/)
- [Fortify SAST](https://www.microfocus.com/documentation/fortify-static-code-analyzer-and-tools/)
- [Fortify SC DAST](https://www.microfocus.com/documentation/fortify-ScanCentral-DAST/)
- [Fortify Azure DevOps Extension Documentation](https://www.microfocus.com/documentation/fortify-azure-devops-extension/)
- [MicroFocus Fortify for Azure DevOps](https://marketplace.visualstudio.com/items?itemName=fortifyvsts.hpe-security-fortify-vsts)
- [Trivy](https://github.com/aquasecurity/trivy)
- [DefectDojo Documentation](https://docs.defectdojo.com)

---

## 🙌 Conclusion

This end-to-end DevSecOps pipeline demonstrates a resilient and automated approach to secure software delivery. By seamlessly integrating industry-standard security tools and a custom Azure DevOps extension, we established a pipeline that ensures continuous security validation and effective vulnerability management throughout the development lifecycle.

We invite your feedback, suggestions, and collaboration ideas—let’s continue advancing secure software practices together.

**🔗 Connect with us:**

* [Hossam Ibraheem on LinkedIn](https://www.linkedin.com/in/hossamibraheem/)
* [Michael Milad on LinkedIn](https://www.linkedin.com/in/michaelmiladcs/)
* Explore the project on [GitHub](https://github.com/0xCoolSAM/0xCoolSAM.github.io/)