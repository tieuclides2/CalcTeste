pipeline {
  agent { label 'delphi-qa' }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    BDS = 'C:\\DelphiCompiler\\23.0'

    // Subpastas no workspace
    PROJ_DIR  = 'CalcProject'
    TEST_DIR  = 'CalcProjectTests'

    // Runner DUnitX (repo de testes)
    TEST_DPR  = 'CalcProjectTests\\Project1.dpr'

    // Saída do runner
    OUT_DIR   = 'Win32\\Release'
    OUT_EXE   = 'Win32\\Release\\Project1.exe'   // nome padrão do seu Project1
  }

  stages {

    stage('Checkout (2 repos)') {
      steps {
        dir(env.PROJ_DIR) {
          checkout([$class: 'GitSCM',
            branches: [[name: '*/main']],
            userRemoteConfigs: [[
              url: 'https://github.com/tieuclides2/CalcProject.git'
              // , credentialsId: 'SEU_CRED_ID' // se precisar
            ]]
          ])
        }

        dir(env.TEST_DIR) {
          checkout([$class: 'GitSCM',
            branches: [[name: '*/main']],
            userRemoteConfigs: [[
              url: 'https://github.com/tieuclides2/CalcTeste.git'
              // , credentialsId: 'SEU_CRED_ID' // se precisar
            ]]
          ])
        }
      }
    }

    stage('Build tests (dcc32)') {
      steps {
        bat """
          @echo on
          cd /d "%WORKSPACE%"

          call "%BDS%\\bin\\rsvars.bat"
          if errorlevel 1 exit /b 1

          rem Pasta de saida
          if not exist "%OUT_DIR%" mkdir "%OUT_DIR%"

          rem Evita rodar exe antigo
          if exist "%OUT_EXE%" del /q "%OUT_EXE%"

          rem Compila o runner de testes e inclui o repo do projeto no Search Path (-U)
          dcc32 --no-config -B -Q -DRELEASE ^
            -E".\\%OUT_DIR%" -NU".\\%OUT_DIR%" ^
            -U"%BDS%\\lib\\win32\\release";"%BDS%\\lib\\win32\\release\\en";"%WORKSPACE%\\%PROJ_DIR%" ^
            -I"%BDS%\\include" ^
            -R"%BDS%\\lib\\win32\\release";"%BDS%\\lib\\win32\\release\\en" ^
            "%TEST_DPR%"

          if errorlevel 1 exit /b 1

          if not exist "%OUT_EXE%" (
            echo ERROR: runner nao foi gerado: %OUT_EXE%
            exit /b 1
          )

          dir "%OUT_EXE%"
        """
      }
    }

    stage('Run tests') {
      steps {
        bat """
          @echo on
          cd /d "%WORKSPACE%"
          "%OUT_EXE%"
          exit /b %ERRORLEVEL%
        """
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**\\Win32\\Release\\**\\*', fingerprint: true
      // sem cleanWs (mantém workspace)
    }
  }
}

