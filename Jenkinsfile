pipeline {
  agent any
  options { timestamps() }

  environment {
    DELPHI_HOME = 'C:\\DelphiCompiler\\23.0'
    // Nome do host do servidor (pra fallback UNC)
    SERVER_HOST = 'testes-pc'
  }

  stages {

    stage('Checkout (2 repos)') {
      steps {
        deleteDir()
        dir('CalcProject') { git branch: 'main', url: 'https://github.com/tieuclides2/CalcProject.git' }
        dir('CalcTeste')   { git branch: 'main', url: 'https://github.com/tieuclides2/CalcTeste.git' }
      }
    }

    stage('Prepare environment (diagnóstico)') {
      steps {
        bat '''
          @echo on
          echo === WHOAMI / CONTEXTO ===
          whoami
          echo.
          echo === Delphi Env ===
          call "C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat"
          where msbuild
          where dcc32
          echo.
          echo === Verificando se o Jenkins enxerga C:\\DelphiCompiler ===
          if exist "C:\\DelphiCompiler" (
            echo OK: C:\\DelphiCompiler existe
            dir "C:\\DelphiCompiler"
          ) else (
            echo ERRO: Jenkins NAO enxerga C:\\DelphiCompiler
          )
        '''
      }
    }

    stage('Resolver path do WebCharts') {
      steps {
        script {
          // Vamos descobrir o path real que o Jenkins consegue acessar
          def out = bat(
            returnStdout: true,
            script: '''
              @echo off
              setlocal EnableExtensions EnableDelayedExpansion

              set "C1=C:\\DelphiCompiler\\Componentes\\TBGWebCharts"
              set "C2=C:\\DelphiCompiler\\Componentes\\TBGWebCharts\\TBGWebCharts"
              set "C3=\\\\testes-pc\\DelphiCompiler\\Componentes\\TBGWebCharts"

              for %%P in ("%C1%" "%C2%" "%C3%") do (
                if exist "%%~P\\View.WebCharts.pas" (
                  echo FOUND=%%~P
                  exit /b 0
                )
              )

              echo FOUND=
              echo --- DEBUG LISTINGS ---
              echo [1] dir C:\\DelphiCompiler\\Componentes
              dir "C:\\DelphiCompiler\\Componentes" 2>nul
              echo [2] dir C:\\DelphiCompiler\\Componentes\\TBGWebCharts
              dir "C:\\DelphiCompiler\\Componentes\\TBGWebCharts" 2>nul
              echo [3] dir \\\\testes-pc\\DelphiCompiler\\Componentes\\TBGWebCharts
              dir "\\\\testes-pc\\DelphiCompiler\\Componentes\\TBGWebCharts" 2>nul
              exit /b 1
            '''
          ).trim()

          def foundLine = out.readLines().find { it.startsWith('FOUND=') }
          def found = foundLine?.substring('FOUND='.length())?.trim()

          if (!found) {
            error("Nao foi possivel localizar View.WebCharts.pas. Saida:\n${out}")
          }

          env.WEBCHARTS_DIR = found
          echo "WEBCHARTS_DIR resolvido: ${env.WEBCHARTS_DIR}"
        }
      }
    }

    stage('Build APP (CalcProject)') {
      steps {
        dir('CalcProject') {
          bat '''
            @echo on
            call "%DELPHI_HOME%\\bin\\rsvars.bat"

            set "PLATFORM=Win32"
            set "CONFIG=Release"
            echo WEBCHARTS_DIR=%WEBCHARTS_DIR%

            msbuild Calc.dproj /t:Build ^
              /p:Config=%CONFIG% /p:Platform=%PLATFORM% ^
              /p:DCC_UnitSearchPath="%WEBCHARTS_DIR%" ^
              /p:DCC_IncludePath="%WEBCHARTS_DIR%" ^
              /p:DCC_OutputDir="Win32\\%CONFIG%"

            if errorlevel 1 exit /b 1
          '''
        }
      }
    }

    stage('Build TESTS (CalcTeste)') {
      steps {
        dir('CalcTeste') {
          bat '''
            @echo on
            call "%DELPHI_HOME%\\bin\\rsvars.bat"

            set "PLATFORM=Win32"
            set "CONFIG=Release"
            set "APP_SRC=%WORKSPACE%\\CalcProject"

            echo WEBCHARTS_DIR=%WEBCHARTS_DIR%
            echo APP_SRC=%APP_SRC%

            msbuild Project1.dproj /t:Build ^
              /p:Config=%CONFIG% /p:Platform=%PLATFORM% ^
              /p:DCC_UnitSearchPath="%APP_SRC%;%WEBCHARTS_DIR%" ^
              /p:DCC_IncludePath="%APP_SRC%;%WEBCHARTS_DIR%" ^
              /p:DCC_OutputDir="Win32\\%CONFIG%"

            if errorlevel 1 exit /b 1
          '''
        }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        dir('CalcTeste') {
          bat '''
            @echo on
            set "CONFIG=Release"
            set "TEST_EXE=%WORKSPACE%\\CalcTeste\\Win32\\%CONFIG%\\Project1.exe"

            if not exist "%TEST_EXE%" (
              echo Test EXE nao encontrado: %TEST_EXE%
              dir "%WORKSPACE%\\CalcTeste\\Win32\\%CONFIG%"
              exit /b 1
            )

            "%TEST_EXE%"
            exit /b %ERRORLEVEL%
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts allowEmptyArchive: true,
        artifacts: 'CalcProject\\Win32\\**\\*.exe, CalcProject\\Win32\\**\\*.dll, CalcProject\\Win32\\**\\*.map',
        fingerprint: true

      archiveArtifacts allowEmptyArchive: true,
        artifacts: 'CalcTeste\\Win32\\**\\*.exe, CalcTeste\\**\\*.xml, CalcTeste\\**\\*.log',
        fingerprint: true
    }
  }
}
