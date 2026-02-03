pipeline {
  agent any
  options { timestamps() }

  environment {
    DELPHI_HOME = 'C:\\DelphiCompiler\\23.0'
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
          if not exist "C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat" (
            echo ERRO: rsvars.bat nao encontrado em C:\\DelphiCompiler\\23.0\\bin
            exit /b 1
          )
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
            exit /b 1
          )
        '''
      }
    }

    stage('Resolver path do WebCharts') {
      steps {
        script {
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

    stage('Resolver path do ACBr + Synapse (synalist)') {
      steps {
        script {
          def out = bat(
            returnStdout: true,
            script: '''
              @echo off
              setlocal EnableExtensions EnableDelayedExpansion

              set "A1=C:\\DelphiCompiler\\Componentes\\ACBr"
              set "A2=\\\\testes-pc\\DelphiCompiler\\Componentes\\ACBr"

              set "FOUND_ACBR="
              for %%P in ("%A1%" "%A2%") do (
                if exist "%%~P\\Fontes\\ACBrComum\\ACBrBase.pas" (
                  set "FOUND_ACBR=%%~P"
                  goto :acbr_ok
                )
              )
              :acbr_ok

              if "%FOUND_ACBR%"=="" (
                echo FOUND_ACBR=
                echo FOUND_SYN=
                echo ERRO: Nao achei ACBrBase.pas em Fontes\\ACBrComum
                exit /b 1
              )

              rem caminho confirmado por você:
              set "S1=%FOUND_ACBR%\\Fontes\\Terceiros\\synalist"
              set "FOUND_SYN="
              if exist "%S1%\\synautil.pas" (
                set "FOUND_SYN=%S1%"
              )

              echo FOUND_ACBR=%FOUND_ACBR%
              echo FOUND_SYN=%FOUND_SYN%
              exit /b 0
            '''
          ).trim()

          def lines = out.readLines()
          def acbrLine = lines.find { it.startsWith('FOUND_ACBR=') }
          def synLine  = lines.find { it.startsWith('FOUND_SYN=') }

          def acbrDir = acbrLine?.substring('FOUND_ACBR='.length())?.trim()
          def synDir  = synLine?.substring('FOUND_SYN='.length())?.trim()

          if (!acbrDir) {
            error("Nao foi possivel localizar ACBrBase.pas. Saida:\n${out}")
          }
          if (!synDir) {
            error("Nao foi possivel localizar synautil.pas em Fontes\\Terceiros\\synalist. Saida:\n${out}")
          }

          env.ACBR_DIR   = acbrDir
          env.ACBR_FONTS = "${env.ACBR_DIR}\\Fontes"
          env.SYN_DIR    = synDir

          env.ACBR_PATH =
            "${env.ACBR_FONTS}\\ACBrComum;" +
            "${env.ACBR_FONTS}\\ACBrDiversos;" +
            "${env.ACBR_FONTS}\\ACBrTCP;" +
            "${env.SYN_DIR}"

          echo "ACBR_DIR resolvido: ${env.ACBR_DIR}"
          echo "ACBR_FONTS: ${env.ACBR_FONTS}"
          echo "SYN_DIR: ${env.SYN_DIR}"
          echo "ACBR_PATH: ${env.ACBR_PATH}"
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
            echo ACBR_PATH=%ACBR_PATH%

            msbuild Calc.dproj /t:Build ^
              /p:Config=%CONFIG% /p:Platform=%PLATFORM% ^
              /p:DCC_UnitSearchPath="%WEBCHARTS_DIR%;%ACBR_PATH%" ^
              /p:DCC_IncludePath="%WEBCHARTS_DIR%;%ACBR_PATH%" ^
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
            echo ACBR_PATH=%ACBR_PATH%
            echo APP_SRC=%APP_SRC%

            msbuild Project1.dproj /t:Build ^
              /p:Config=%CONFIG% /p:Platform=%PLATFORM% ^
              /p:DCC_UnitSearchPath="%APP_SRC%;%WEBCHARTS_DIR%;%ACBR_PATH%" ^
              /p:DCC_IncludePath="%APP_SRC%;%WEBCHARTS_DIR%;%ACBR_PATH%" ^
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
