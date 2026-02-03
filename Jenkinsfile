// Jenkinsfile (Windows agent) - Delphi + vendors + FastReport + cache global

@NonCPS
String extractMissingUnit(String logText) {
  if (logText == null) return null
  def m = (logText =~ /error\s+F2613:\s+Unit\s+'([^']+)'\s+not\s+found/i)
  return (m && m.size() > 0) ? m[0][1] : null
}

String q(String p) { return "\"${p}\"" } // quote helper for bat/msbuild

pipeline {
  agent { label 'delphi-qa' }

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    // Delphi
    DELPHI_ROOT = 'C:\\DelphiCompiler\\23.0'
    RSVARS      = 'C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat'

    // Vendors base
    COMPONENTES = 'C:\\DelphiCompiler\\Componentes'

    // Cache global
    JENKINS_CACHE = 'C:\\Jenkins\\delphi-qa\\JenkinsCache'
    DCU_CACHE     = 'C:\\Jenkins\\delphi-qa\\JenkinsCache\\DCU'

    // Build defaults
    CFG  = 'Release'
    PLAT = 'Win32'
  }

  stages {
    stage('Checkout (2 repos)') {
      steps {
        deleteDir()

        dir('CalcProject') {
          git url: 'https://github.com/tieuclides2/CalcProject.git', branch: 'main'
        }

        dir('CalcTeste') {
          git url: 'https://github.com/tieuclides2/CalcTeste.git', branch: 'main'
        }
      }
    }

    stage('Prepare environment (diagnóstico)') {
      steps {
        bat """
          echo === WHOAMI / CONTEXTO ===
          whoami
          echo.

          echo === Delphi Env ===
          if not exist ${q(env.RSVARS)} (
            echo ERRO: rsvars.bat nao encontrado em ${env.RSVARS}
            exit /b 1
          )
          call ${q(env.RSVARS)}
          where msbuild
          where dcc32
          echo.

          echo === Componentes ===
          if not exist ${q(env.COMPONENTES)} (
            echo ERRO: Pasta Componentes nao existe: ${env.COMPONENTES}
            exit /b 1
          )
          dir ${q(env.COMPONENTES)} /ad
          echo.

          echo === Cache ===
          if not exist ${q(env.JENKINS_CACHE)} mkdir ${q(env.JENKINS_CACHE)}
          if not exist ${q(env.DCU_CACHE)} mkdir ${q(env.DCU_CACHE)}
          dir ${q(env.JENKINS_CACHE)} /ad
        """
      }
    }

    stage('Resolver roots (vendors + FastReport)') {
      steps {
        script {
          // Vendors existentes no seu servidor
          def webChartsDir = "${env.COMPONENTES}\\TBGWebCharts"
          def acbrDir      = "${env.COMPONENTES}\\ACBr"
          def bceditorDir  = "${env.COMPONENTES}\\BCEditor"
          def redsisDir    = "${env.COMPONENTES}\\RedsisComponents"

          // FastReport (pela sua estrutura no print)
          def frBase   = "${env.COMPONENTES}\\Fast Reports\\VCL\\2025.2.2"
          def frSrc    = "${frBase}\\Sources"
          def frLib    = "${frBase}\\LibRS29"

          // ACBr subpaths (o que você já vinha usando)
          def acbrPath = [
            "${acbrDir}\\Fontes\\ACBrComum",
            "${acbrDir}\\Fontes\\ACBrDiversos",
            "${acbrDir}\\Fontes\\ACBrTCP",
            "${acbrDir}\\Fontes\\Terceiros\\synalist",
            "${acbrDir}\\Fontes\\Terceiros\\FastStringReplace",
            "${acbrDir}\\Fontes\\Terceiros\\GZIPUtils",
            "${acbrDir}\\Fontes\\Terceiros"
          ].join(';')

          // FastReport path: fonte + libs (DCU)
          // (mesmo que frLib não tenha DCU, deixar no path não atrapalha)
          def frPath = [
            frSrc,
            frLib
          ].join(';')

          // Path inicial do compilador (vendor paths “known good”)
          // (ordem importa: preferir DCU/lib antes de sources, se houver)
          env.UNIT_PATH = [
            webChartsDir,
            acbrPath,
            frPath
          ].join(';')

          // Roots para auto-inject (procura UnitName.pas dentro desses lugares)
          env.VENDOR_ROOTS = [
            webChartsDir,
            acbrDir,
            "${acbrDir}\\Fontes",
            "${acbrDir}\\Fontes\\Terceiros",
            bceditorDir,
            redsisDir,
            frBase,
            frSrc,
            frLib
          ].join(';')

          echo "WEBCHARTS_DIR: ${webChartsDir}"
          echo "ACBR_DIR: ${acbrDir}"
          echo "BCEDITOR_DIR: ${bceditorDir}"
          echo "REDSIS_DIR: ${redsisDir}"
          echo "FASTREPORT_BASE: ${frBase}"
          echo "FASTREPORT_SOURCES: ${frSrc}"
          echo "FASTREPORT_LIB: ${frLib}"
          echo "UNIT_PATH: ${env.UNIT_PATH}"
          echo "VENDOR_ROOTS: ${env.VENDOR_ROOTS}"
        }
      }
    }

    stage('Build APP (CalcProject) - auto-inject') {
      steps {
        dir('CalcProject') {
          script {
            int maxTries = 12

            for (int i = 1; i <= maxTries; i++) {
              echo "=== Build tentativa ${i}/${maxTries} ==="
              echo "UnitSearchPath atual: ${env.UNIT_PATH}"

              def logFile = "msbuild_calc_try${i}.log"

              def rc = bat(returnStatus: true, script: """
                @echo on
                call ${q(env.RSVARS)}
                echo === MSBUILD Calc.dproj CFG=${env.CFG} PLAT=${env.PLAT} ===

                rem Opcional: separar DCU do projeto (se seu MSBuild/Delphi respeitar)
                rem Se não respeitar, apenas ignore: não quebra o build.
                set DCU_OUT=${q(env.DCU_CACHE)}\\CalcProject\\${env.PLAT}\\${env.CFG}
                if not exist "%DCU_OUT%" mkdir "%DCU_OUT%"

                msbuild "Calc.dproj" /t:Build ^
                  /p:Config=${env.CFG} /p:Platform=${env.PLAT} ^
                  /p:DCC_UnitSearchPath=${q(env.UNIT_PATH)} ^
                  /p:DCC_IncludePath=${q(env.UNIT_PATH)} ^
                  /p:DCC_OutputDir=${q("${env.PLAT}\\\\${env.CFG}")} ^
                  /p:DCC_DcuOutput=${q("%DCU_OUT%")} ^
                  /fl /flp:logfile=${logFile};verbosity=diagnostic

                exit /b %errorlevel%
              """)

              def logText = readFile(logFile)
              def missing = extractMissingUnit(logText)

              if (rc == 0) {
                echo "Build OK."
                break
              }

              if (missing == null) {
                error "Falha no build, mas não consegui extrair 'error F2613' do log (${logFile})."
              }

              echo "Unit faltando: ${missing}"

              // procura UnitName.pas nos roots e injeta a pasta encontrada no UNIT_PATH
              def foundDir = bat(returnStdout: true, script: """
                @echo off
                setlocal enabledelayedexpansion
                set UNIT=${missing}
                set ROOTS=${env.VENDOR_ROOTS}

                for %%R in (!ROOTS!) do (
                  for /f "delims=" %%F in ('dir /s /b "%%~R\\!UNIT!.pas" 2^>nul') do (
                    for %%D in ("%%~dpF.") do (
                      echo %%~fD
                      exit /b 0
                    )
                  )
                )
                exit /b 1
              """).trim()

              if (!foundDir) {
                error "Não encontrei ${missing}.pas em VENDOR_ROOTS. Adicione o root do vendor no Jenkinsfile."
              }

              // injeta apenas a pasta (sem repetir)
              if (!env.UNIT_PATH.toLowerCase().contains(foundDir.toLowerCase())) {
                env.UNIT_PATH = "${env.UNIT_PATH};${foundDir}"
                echo "Injetado no UnitSearchPath: ${foundDir}"
              } else {
                error "Encontrei ${missing}.pas, mas a pasta já estava no UnitSearchPath e mesmo assim falhou. Verifique conflito de versões/DCU."
              }

              if (i == maxTries) {
                error "Excedeu tentativas (${maxTries}) no build do app."
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
            // Ajuste aqui para o NOME REAL do .dproj de testes
            def testDproj = 'CalcTeste.dproj'

            def exists = fileExists(testDproj)
            if (!exists) {
              error "Não achei ${testDproj} em CalcTeste. Ajuste o nome do projeto de testes no Jenkinsfile."
            }

            int maxTries = 12

            for (int i = 1; i <= maxTries; i++) {
              echo "=== Build TEST tentativa ${i}/${maxTries} ==="

              def logFile = "msbuild_tests_try${i}.log"

              def rc = bat(returnStatus: true, script: """
                @echo on
                call ${q(env.RSVARS)}

                set DCU_OUT=${q(env.DCU_CACHE)}\\CalcTeste\\${env.PLAT}\\${env.CFG}
                if not exist "%DCU_OUT%" mkdir "%DCU_OUT%"

                msbuild "${testDproj}" /t:Build ^
                  /p:Config=${env.CFG} /p:Platform=${env.PLAT} ^
                  /p:DCC_UnitSearchPath=${q(env.UNIT_PATH)} ^
                  /p:DCC_IncludePath=${q(env.UNIT_PATH)} ^
                  /p:DCC_OutputDir=${q("${env.PLAT}\\\\${env.CFG}")} ^
                  /p:DCC_DcuOutput=${q("%DCU_OUT%")} ^
                  /fl /flp:logfile=${logFile};verbosity=diagnostic

                exit /b %errorlevel%
              """)

              def logText = readFile(logFile)
              def missing = extractMissingUnit(logText)

              if (rc == 0) {
                echo "Build TESTS OK."
                break
              }

              if (missing == null) {
                error "Falha nos testes, mas não consegui extrair 'error F2613' do log (${logFile})."
              }

              echo "Unit faltando (tests): ${missing}"

              def foundDir = bat(returnStdout: true, script: """
                @echo off
                setlocal enabledelayedexpansion
                set UNIT=${missing}
                set ROOTS=${env.VENDOR_ROOTS}

                for %%R in (!ROOTS!) do (
                  for /f "delims=" %%F in ('dir /s /b "%%~R\\!UNIT!.pas" 2^>nul') do (
                    for %%D in ("%%~dpF.") do (
                      echo %%~fD
                      exit /b 0
                    )
                  )
                )
                exit /b 1
              """).trim()

              if (!foundDir) {
                error "Não encontrei ${missing}.pas em VENDOR_ROOTS (tests)."
              }

              if (!env.UNIT_PATH.toLowerCase().contains(foundDir.toLowerCase())) {
                env.UNIT_PATH = "${env.UNIT_PATH};${foundDir}"
                echo "Injetado no UnitSearchPath (tests): ${foundDir}"
              } else {
                error "Encontrei ${missing}.pas (tests), mas já estava no UnitSearchPath e mesmo assim falhou."
              }

              if (i == maxTries) {
                error "Excedeu tentativas (${maxTries}) no build dos testes."
              }
            }
          }
        }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        dir('CalcTeste') {
          // Ajuste o nome do exe de testes gerado pelo seu projeto
          // Ex.: Win32\Release\CalcTeste.exe ou algo como Tests.exe
          bat """
            @echo on
            set TEST_EXE=Win32\\Release\\CalcTeste.exe
            if not exist "%TEST_EXE%" (
              echo ERRO: nao achei o executavel de testes: %TEST_EXE%
              echo Ajuste o caminho/nome no Jenkinsfile.
              exit /b 1
            )
            "%TEST_EXE%" --run
          """
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/Win32/**, **/*.log', allowEmptyArchive: true
    }
  }
}
