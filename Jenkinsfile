pipeline {
  agent any
  options { timestamps() }

  environment {
    DELPHI_HOME    = 'C:\\DelphiCompiler\\23.0'
    COMPONENTS     = 'C:\\DelphiCompiler\\Componentes'

    // Evita colisão com variável PLATFORM do serviço/sistema
    BUILD_PLATFORM = 'Win32'
    BUILD_CONFIG   = 'Release'
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
          if not exist "%DELPHI_HOME%\\bin\\rsvars.bat" (
            echo ERRO: rsvars.bat nao encontrado em %DELPHI_HOME%\\bin
            exit /b 1
          )
          call "%DELPHI_HOME%\\bin\\rsvars.bat"
          where msbuild
          where dcc32
          echo.

          echo === Componentes ===
          if not exist "%COMPONENTS%" (
            echo ERRO: Pasta Componentes nao existe: %COMPONENTS%
            exit /b 1
          )
          dir "%COMPONENTS%" /ad
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

              set "C1=%COMPONENTS%\\TBGWebCharts"
              set "C2=%COMPONENTS%\\TBGWebCharts\\TBGWebCharts"

              for %%P in ("%C1%" "%C2%") do (
                if exist "%%~P\\View.WebCharts.pas" (
                  echo FOUND=%%~P
                  exit /b 0
                )
              )

              echo FOUND=
              exit /b 1
            '''
          ).trim()

          def foundLine = out.readLines().find { it.startsWith('FOUND=') }
          def found = foundLine?.substring('FOUND='.length())?.trim()
          if (!found) error("Nao foi possivel localizar View.WebCharts.pas.\n${out}")

          env.WEBCHARTS_DIR = found
          echo "WEBCHARTS_DIR resolvido: ${env.WEBCHARTS_DIR}"
        }
      }
    }

    stage('Resolver roots (ACBr / BCEditor / Redsis)') {
      steps {
        script {
          env.ACBR_DIR   = "${env.COMPONENTS}\\ACBr"
          env.BCED_DIR   = "${env.COMPONENTS}\\BCEditor"
          env.REDSIS_DIR = "${env.COMPONENTS}\\RedsisComponents"

          def acbrFonts = "${env.ACBR_DIR}\\Fontes"
          def synalist  = "${acbrFonts}\\Terceiros\\synalist"
          def fastStr   = "${acbrFonts}\\Terceiros\\FastStringReplace"

          env.ACBR_PATH = [
            "${acbrFonts}\\ACBrComum",
            "${acbrFonts}\\ACBrDiversos",
            "${acbrFonts}\\ACBrTCP",
            synalist,
            fastStr
          ].join(';')

          env.VENDOR_ROOTS = [
            env.WEBCHARTS_DIR,
            env.ACBR_DIR,
            env.BCED_DIR,
            env.REDSIS_DIR
          ].join(';')

          echo "ACBR_DIR: ${env.ACBR_DIR}"
          echo "BCEDITOR_DIR: ${env.BCED_DIR}"
          echo "REDSIS_DIR: ${env.REDSIS_DIR}"
          echo "ACBR_PATH: ${env.ACBR_PATH}"
          echo "VENDOR_ROOTS: ${env.VENDOR_ROOTS}"
        }
      }
    }

    stage('Build APP (CalcProject)') {
      steps {
        dir('CalcProject') {
          script {
            buildWithAutoInject('Calc.dproj', "${env.WEBCHARTS_DIR};${env.ACBR_PATH}", env.VENDOR_ROOTS)
          }
        }
      }
    }

    stage('Build TESTS (CalcTeste)') {
      steps {
        dir('CalcTeste') {
          script {
            def appSrc = "${env.WORKSPACE}\\CalcProject"
            buildWithAutoInject('Project1.dproj', "${appSrc};${env.WEBCHARTS_DIR};${env.ACBR_PATH}", env.VENDOR_ROOTS)
          }
        }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        dir('CalcTeste') {
          bat '''
            @echo on
            set "TEST_EXE=%WORKSPACE%\\CalcTeste\\Win32\\%BUILD_CONFIG%\\Project1.exe"

            if not exist "%TEST_EXE%" (
              echo Test EXE nao encontrado: %TEST_EXE%
              dir "%WORKSPACE%\\CalcTeste\\Win32\\%BUILD_CONFIG%"
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

/**
 * Roda o msbuild em loop.
 * Se falhar com F2613 Unit 'X' not found, procura X.pas em vendorRoots e injeta o dir automaticamente.
 */
def buildWithAutoInject(String dproj, String baseUnitPath, String vendorRoots) {
  int maxTries = 12
  def injected = [] as Set

  for (int i = 1; i <= maxTries; i++) {
    def injectedPath = injected ? (';' + injected.join(';')) : ''
    def unitPath = "${baseUnitPath}${injectedPath}"

    echo "=== Build tentativa ${i}/${maxTries} ==="
    echo "UnitSearchPath atual: ${unitPath}"

    // Importante: NÃO pode falhar o step bat, senão não conseguimos parsear e iterar.
    def out = bat(returnStdout: true, script: """
      @echo off
      setlocal EnableExtensions EnableDelayedExpansion

      call "%DELPHI_HOME%\\bin\\rsvars.bat"

      set "CFG=%BUILD_CONFIG%"
      set "PLAT=%BUILD_PLATFORM%"

      set "LOG=msbuild_${i}.log"

      msbuild "${dproj}" /t:Build ^
        /p:Config=!CFG! /p:Platform=!PLAT! ^
        /p:DCC_UnitSearchPath="${unitPath}" ^
        /p:DCC_IncludePath="${unitPath}" ^
        /p:DCC_OutputDir="Win32\\\\!CFG!" > "!LOG!" 2>&1

      set "RC=!ERRORLEVEL!"

      type "!LOG!"
      echo __RC__=!RC!

      exit /b 0
    """).trim()

    def rcLine = out.readLines().reverse().find { it.startsWith('__RC__=') }
    def rc = rcLine ? (rcLine.substring('__RC__='.length()) as int) : 999

    if (rc == 0) {
      echo "Build OK."
      return
    }

    // Tenta extrair unit faltando
    def m = (out =~ /error F2613: Unit '([^']+)' not found\./)
    if (!m.find()) {
      error("Build falhou (RC=${rc}) mas não é F2613 (Unit not found).\n\n${out}")
    }

    def unitName = m.group(1)
    echo "Unit faltando: ${unitName}"

    def foundDir = bat(returnStdout: true, script: """
      @echo off
      setlocal EnableExtensions EnableDelayedExpansion

      set "TARGET=${unitName}.pas"
      set "ROOTS=${vendorRoots}"

      for %%R in (!ROOTS:;= !) do (
        if exist "%%~R" (
          for /f "delims=" %%F in ('dir /s /b "%%~R\\!TARGET!" 2^>nul') do (
            echo %%~dpF
            exit /b 0
          )
        )
      )

      exit /b 1
    """).trim()

    if (!foundDir) {
      error(
        "Nao achei ${unitName}.pas dentro de VENDOR_ROOTS.\n" +
        "Se essa dependencia vier só como DCU/DCP, precisa adicionar a pasta de DCU no search path.\n\n" +
        "Log do build:\n${out}"
      )
    }

    foundDir = foundDir.replaceAll(/[\\\\\\s]+$/, '')

    if (!injected.contains(foundDir)) {
      injected.add(foundDir)
      echo "Injetando no path: ${foundDir}"
    } else {
      error("Já injetei ${foundDir} e o erro persiste. Log:\n\n${out}")
    }
  }

  error("Excedeu tentativas de auto-inject (${maxTries}).")
}
