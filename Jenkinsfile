pipeline {
  agent any

  tools {
    // 与「系统管理 → 全局工具配置」里 Node 名称一致，如 "Node 20"
    nodejs 'Node 24'
  }

  environment {
    // 若使用 pnpm，可取消下一行注释，并确保 Jenkins 能安装/使用 pnpm
    PATH = "/usr/local/bin:${env.PATH}"
  }

  stages {
    stage('拉取代码') {
      steps {
        checkout scm
      }
    }

    stage('安装依赖') {
      steps {
        sh 'node -v && npm -v'
        // 使用 npm（默认）
        // sh 'npm ci --prefer-offline --no-audit'
        // 若项目用 pnpm，可改为：
        sh 'npm install -g pnpm && pnpm install --frozen-lockfile'
      }
    }

    stage('构建') {
      steps {
        sh 'npm run build'
      }
    }

    stage('归档产物') {
      steps {
        // 归档 .next 与 package.json 等，便于后续部署或备份
        archiveArtifacts artifacts: '.next/**,package.json,package-lock.json', fingerprint: true
      }
    }
  }

  post {
    success {
      echo '构建成功'
    }
    failure {
      echo '构建失败'
    }
    always {
      cleanWs(deleteDirs: true, notFailBuild: true)
      // 若希望保留工作空间便于排查，可注释上一行
    }
  }
}
