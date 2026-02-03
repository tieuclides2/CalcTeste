pipeline {
  agent { label 'delphi-qa' }

  options {
    timestamps()
    skipDefaultCheckout(true)
  }

  environment {
    // Delphi
    DELPHI_ROOT      = 'C:\\DelphiCompiler\\23.0'
    RSVARS_BAT       = 'C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat'

    // Terceiros
    COMPONENTS_ROOT  = 'C:\\DelphiCompiler\\Componentes'

    // Repos
    APP_REPO_URL     = 'https://github.com/tieuclides2/CalcProject.git'
    TEST_REPO_URL    = 'https://github.com/tieuclides2/CalcTeste.git'

    // Build default
    CFG              = 'Release'
    PLAT             = 'Win32'
  }

  stages {

    stage('Checkout (2 repos)') {
      steps {
        deleteDir()
        dir('CalcProject') {
          git url: env.APP_REPO_URL, branch: 'main'
        }
        dir('CalcTeste') {
          git url: env.TEST_REPO_URL, branch: 'main'
        }
      }
    }

    stage('Prepare environment (diagnóstico)') {
      steps {
        bat """
          @echo off
          echo === WHOAMI / CONTEXTO ===
          whoami
          echo.

          echo === Delphi Env ===
          if not exist "${RSVARS_BAT}" (
            echo ERRO: rsvars.bat nao encontrado em ${RSVARS_BAT}
            exit /b 1
          )
          call "${RSVARS_BAT}"
          where msbuild
          where dcc32
          echo.

          echo === Componentes ===
          if not exist "${COMPONENTS_ROOT}" (
            echo ERRO: Pasta Componentes nao existe: ${COMPONENTS_ROOT}
            exit /b 1
          )
          dir "${COMPONENTS_ROOT}" /ad
        """
      }
    }

    stage('Resolver roots (vendors + ACBr)') {
      steps {
        script {
          // Vendors “raiz”
          env.WEBCHARTS_DIR = "${env.COMPONENTS_ROOT}\\TBGWebCharts"
          env.ACBR_DIR      = "${env.COMPONENTS_ROOT}\\ACBr"
          env.BCEDITOR_DIR  = "${env.COMPONENTS_ROOT}\\BCEditor"
          env.REDSIS_DIR    = "${env.COMPONENTS_ROOT}\\RedsisComponents"

          // ACBr “miolo”
          def acbrFonts     = "${env.ACBR_DIR}\\Fontes"
          def acbrTerceiros = "${acbrFonts}\\Terceiros"

          // Search-path base (o que você já vinha usando)
          env.ACBR_PATH = [
            "${acbrFonts}\\ACBrComum",
            "${acbrFonts}\\ACBrDiversos",
            "${acbrFonts}\\ACBrTCP",

            // Terceiros já identificados no seu ambiente
            "${acbrTerceiros}\\synalist",
            "${acbrTerceiros}\\FastStringReplace",
            "${acbrTerceiros}\\GZIPUtils",

            // (importante) Terceiros “root” também, pra cobrir futuros casos
            "${acbrTerceiros}"
          ].join(';')

          // Roots de busca (para o auto-inject achar UnitName.pas/.dcu)
          env.VENDOR_ROOTS = [
            env.WEBCHARTS_DIR,
            env.ACBR_DIR,
            acbrFonts,
            acbrTerceiros,
            env.BCEDITOR_DIR,
            env.REDSIS_DIR
          ].join(';')

          echo "WEBCHARTS_DIR: ${env.WEBCHARTS_DIR}"
          echo "ACBR_DIR: ${env.ACBR_DIR}"
          echo "BCEDITOR_DIR: ${env.BCEDITOR_DIR}"
          echo "REDSIS_DIR: ${env.REDSIS_DIR}"
          echo "ACBR_PATH: ${env.ACBR_PATH}"
          echo "VENDOR_ROOTS: ${env.VENDOR_ROOTS}"
        }
      }
    }

    stage('Build APP (CalcProject) - auto-inject') {
      steps {
        dir('CalcProject') {
          script {

            // Função: executa msbuild e grava log em arquivo (para parsing via bat)
            def runMsbuildToLog = { String dproj, String unitSearchPath, String logFile ->
              // /flp exige caminho simples (sem espaços) ou aspas; aqui usamos arquivo local do workspace
              bat """
                @echo off
                call "${RSVARS_BAT}"
                set "CFG=${CFG}"
                set "PLAT=${PLAT}"
                echo === MSBUILD ${dproj} CFG=%CFG% PLAT=%PLAT% ===
                echo UnitSearchPath=%~1
                rem roda e salva log completo
                msbuild "${dproj}" /t:Build /p:Config=%CFG% /p:Platform=%PLAT% ^
                  /p:DCC_UnitSearchPath="${unitSearchPath}" ^
                  /p:DCC_IncludePath="${unitSearchPath}" ^
                  /p:DCC_OutputDir="Win32\\\\${CFG}" ^
                  /fl /flp:logfile="${logFile}";verbosity=diagnostic
              """
            }

            // Função: extrai a unit faltante do log do msbuild (sem regex Groovy -> evita NotSerializableException)
            def extractMissingUnit = { String logFile ->
              def out = bat(returnStdout: true, script: """
                @echo off
                setlocal EnableExtensions EnableDelayedExpansion
                set "LOG=${logFile}"

                rem pega a primeira ocorrência de F2613 e extrai entre aspas simples: Unit 'XXXX' not found
                for /f "delims=" %%L in ('findstr /i /c:"error F2613:" "!LOG!" 2^>nul') do (
                  set "LINE=%%L"
                  goto :got
                )
                echo.
                exit /b 0

                :got
                rem Extrai conteúdo entre Unit '  e  ' not found
                set "S=!LINE:*Unit '=!"
                for /f "delims=' tokens=1" %%U in ("!S!") do (
                  echo %%U
                  exit /b 0
                )
                echo.
              """).trim()
              return out
            }

            // Função: procura UnitName.pas OU UnitName.dcu dentro dos roots e devolve a pasta (string)
            def findUnitDir = { String unitName, String vendorRoots ->
              def found = bat(returnStdout: true, script: """
                @echo off
                setlocal EnableExtensions EnableDelayedExpansion

                set "ROOTS=${vendorRoots}"
                set "U=${unitName}"

                rem 1) .pas
                for %%R in (!ROOTS:;= !) do (
                  if exist "%%~R" (
                    for /f "delims=" %%F in ('dir /s /b "%%~R\\!U!.pas" 2^>nul') do (
                      echo %%~dpF
                      exit /b 0
                    )
                  )
                )

                rem 2) .dcu
                for %%R in (!ROOTS:;= !) do (
                  if exist "%%~R" (
                    for /f "delims=" %%F in ('dir /s /b "%%~R\\!U!.dcu" 2^>nul') do (
                      echo %%~dpF
                      exit /b 0
                    )
                  )
                )

                exit /b 1
              """).trim()
              return found
            }

            // --- Auto-inject loop ---
            def basePath = [
              env.WEBCHARTS_DIR,
              env.ACBR_PATH
            ].join(';')

            def unitSearchPath = basePath
            def maxTries = 12

            for (int i = 1; i <= maxTries; i++) {
              echo "=== Build tentativa ${i}/${maxTries} ==="
              echo "UnitSearchPath atual: ${unitSearchPath}"

              def logFile = "msbuild_calcproject_try${i}.log"

              // roda build
              int rc = 0
              try {
                runMsbuildToLog('Calc.dproj', unitSearchPath, logFile)
              } catch (e) {
                rc = 1
              }

              if (rc == 0) {
                echo "Build OK."
                break
              }

              // pega unit faltante
              def missing = extractMissingUnit(logFile)
              if (!missing) {
                error "Falha no build, mas não consegui extrair 'error F2613' do log (${logFile})."
              }

              echo "Unit faltando: ${missing}"

              // procura e injeta
              def dirFound = ''
              try {
                dirFound = findUnitDir(missing, env.VENDOR_ROOTS)
              } catch (e) {
                dirFound = ''
              }

              if (!dirFound) {
                error "Não encontrei ${missing}.pas/.dcu em VENDOR_ROOTS. Ajuste roots ou instale o vendor no servidor."
              }

              // normaliza (remove barra final)
              dirFound = dirFound.replaceAll('\\\\+$','')
              echo "Encontrado em: ${dirFound}"

              // injeta no search path
              if (!unitSearchPath.toLowerCase().contains(dirFound.toLowerCase())) {
                unitSearchPath = unitSearchPath + ';' + dirFound
              } else {
                echo "Pasta já estava no UnitSearchPath, tentando novamente."
              }

              if (i == maxTries) {
                error "Estourou tentativas (${maxTries}). Última unit faltante: ${missing}"
              }
            }
          }
        }
      }
    }

    stage('Build TESTS (CalcTeste) - auto-inject') {
      steps {
        dir('CalcTeste') {
          script {
            // Reaproveita a mesma ideia (mais simples: já começar com o mesmo path base)
            def unitSearchPath = [
              env.WEBCHARTS_DIR,
              env.ACBR_PATH
            ].join(';')

            // Ajuste: se seus testes dependem do app, inclua também a pasta do projeto do app
            // (para units compartilhadas)
            unitSearchPath = unitSearchPath + ';' + "${pwd()}\\..\\CalcProject"

            def maxTries = 12

            def runMsbuildToLog = { String dproj, String usp, String logFile ->
              bat """
                @echo off
                call "${RSVARS_BAT}"
                set "CFG=${CFG}"
                set "PLAT=${PLAT}"
                msbuild "${dproj}" /t:Build /p:Config=%CFG% /p:Platform=%PLAT% ^
                  /p:DCC_UnitSearchPath="${usp}" ^
                  /p:DCC_IncludePath="${usp}" ^
                  /p:DCC_OutputDir="Win32\\\\${CFG}" ^
                  /fl /flp:logfile="${logFile}";verbosity=diagnostic
              """
            }

            def extractMissingUnit = { String logFile ->
              def out = bat(returnStdout: true, script: """
                @echo off
                setlocal EnableExtensions EnableDelayedExpansion
                set "LOG=${logFile}"

                for /f "delims=" %%L in ('findstr /i /c:"error F2613:" "!LOG!" 2^>nul') do (
                  set "LINE=%%L"
                  goto :got
                )
                echo.
                exit /b 0

                :got
                set "S=!LINE:*Unit '=!"
                for /f "delims=' tokens=1" %%U in ("!S!") do (
                  echo %%U
                  exit /b 0
                )
                echo.
              """).trim()
              return out
            }

            def findUnitDir = { String unitName, String vendorRoots ->
              def found = bat(returnStdout: true, script: """
                @echo off
                setlocal EnableExtensions EnableDelayedExpansion

                set "ROOTS=${vendorRoots}"
                set "U=${unitName}"

                for %%R in (!ROOTS:;= !) do (
                  if exist "%%~R" (
                    for /f "delims=" %%F in ('dir /s /b "%%~R\\!U!.pas" 2^>nul') do ( echo %%~dpF & exit /b 0 )
                  )
                )
                for %%R in (!ROOTS:;= !) do (
                  if exist "%%~R" (
                    for /f "delims=" %%F in ('dir /s /b "%%~R\\!U!.dcu" 2^>nul') do ( echo %%~dpF & exit /b 0 )
                  )
                )
                exit /b 1
              """).trim()
              return found
            }

            for (int i = 1; i <= maxTries; i++) {
              echo "=== Build TEST tentativa ${i}/${maxTries} ==="
              def logFile = "msbuild_calcteste_try${i}.log"

              int rc = 0
              try {
                runMsbuildToLog('Calc.dproj', unitSearchPath, logFile)
              } catch (e) {
                rc = 1
              }

              if (rc == 0) {
                echo "Build TEST OK."
                break
              }

              def missing = extractMissingUnit(logFile)
              if (!missing) {
                error "Falha nos testes, mas não consegui extrair 'error F2613' do log (${logFile})."
              }

              echo "Unit faltando (tests): ${missing}"

              def dirFound = ''
              try { dirFound = findUnitDir(missing, env.VENDOR_ROOTS) } catch (e) { dirFound = '' }

              if (!dirFound) {
                error "Não encontrei ${missing}.pas/.dcu em VENDOR_ROOTS (tests)."
              }

              dirFound = dirFound.replaceAll('\\\\+$','')
              echo "Encontrado em: ${dirFound}"

              if (!unitSearchPath.toLowerCase().contains(dirFound.toLowerCase())) {
                unitSearchPath = unitSearchPath + ';' + dirFound
              }

              if (i == maxTries) {
                error "Estourou tentativas (${maxTries}) nos testes. Última unit faltante: ${missing}"
              }
            }
          }
        }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        dir('CalcTeste') {
          bat """
            @echo off
            set "EXE=Win32\\${CFG}\\Calc.exe"
            if not exist "%EXE%" (
              echo ERRO: executavel de testes nao encontrado: %EXE%
              exit /b 1
            )
            "%EXE%"
          """
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/*.log, **/Win32/**', allowEmptyArchive: true
    }
  }
}
