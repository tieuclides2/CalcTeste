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
          def out = bat(returnStdout: true, script: '''
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
            exit /b 1
          ''').trim()

          def found = out.readLines().find { it.startsWith('FOUND=') }?.substring('FOUND='.length())?.trim()
          if (!found) error("Nao foi possivel localizar View.WebCharts.pas. Saida:\n${out}")

          env.WEBCHARTS_DIR = found
          echo "WEBCHARTS_DIR resolvido: ${env.WEBCHARTS_DIR}"
        }
      }
    }

    stage('Resolver ACBr (base)') {
      steps {
        script {
          def out = bat(returnStdout: true, script: '''
            @echo off
            setlocal EnableExtensions EnableDelayedExpansion

            set "A1=C:\\DelphiCompiler\\Componentes\\ACBr"
            set "A2=\\\\testes-pc\\DelphiCompiler\\Componentes\\ACBr"

            set "FOUND_ACBR="
            for %%P in ("%A1%" "%A2%") do (
              if exist "%%~P\\Fontes\\ACBrComum\\ACBrBase.pas" (
                set "FOUND_ACBR=%%~P"
                goto :ok
              )
            )
            :ok
            echo FOUND_ACBR=%FOUND_ACBR%
          ''').trim()

          def acbrDir = out.readLines().find { it.startsWith('FOUND_ACBR=') }?.substring('FOUND_ACBR='.length())?.trim()
          if (!acbrDir) error("Nao foi possivel localizar ACBrBase.pas. Saida:\n${out}")

          env.ACBR_DIR   = acbrDir
          env.ACBR_FONTS = "${env.ACBR_DIR}\\Fontes"

          // paths “fixos” que você já sabe que precisa (ACBrValidator + Synapse)
          def synDir = "${env.ACBR_FONTS}\\Terceiros\\synalist"

          env.ACBR_BASE_PATH =
            "${env.ACBR_FONTS}\\ACBrComum;" +
            "${env.ACBR_FONTS}\\ACBrDiversos;" +
            "${env.ACBR_FONTS}\\ACBrTCP;" +
            "${synDir}"

          echo "ACBR_DIR: ${env.ACBR_DIR}"
          echo "ACBR_BASE_PATH: ${env.ACBR_BASE_PATH}"
        }
      }
    }

    stage('Build APP (CalcProject) - auto inject Units') {
      steps {
        script {
          // roots onde o “auto-find” vai procurar UnitName.pas
          // você pode acrescentar outros vendors aqui no futuro.
          def roots = [
            env.WEBCHARTS_DIR,
            "${env.ACBR_FONTS}\\ACBrComum",
            "${env.ACBR_FONTS}\\ACBrDiversos",
            "${env.ACBR_FONTS}\\ACBrTCP",
            "${env.ACBR_FONTS}\\Terceiros",
            "${env.ACBR_FONTS}" // fallback
          ]

          def extraPaths = [] as Set

          def maxTries = 10
          for (int i = 1; i <= maxTries; i++) {
            def unitSearch = ([env.WEBCHARTS_DIR] + [env.ACBR_BASE_PATH] + extraPaths.toList()).join(';')
            // normaliza ";;" caso ACBR_BASE_PATH já tenha ';' etc.
            unitSearch = unitSearch.replaceAll(/;{2,}/, ';')

            echo "Tentativa ${i}/${maxTries} - UnitSearchPath: ${unitSearch}"

            // compila e captura log
            def log = bat(returnStdout: true, script: """
              @echo off
              setlocal
              call "${env.DELPHI_HOME}\\bin\\rsvars.bat" >nul

              cd /d "%WORKSPACE%\\CalcProject"

              msbuild Calc.dproj /t:Build ^
                /p:Config=Release /p:Platform=Win32 ^
                /p:DCC_UnitSearchPath="${unitSearch}" ^
                /p:DCC_IncludePath="${unitSearch}" ^
                /p:DCC_OutputDir="Win32\\Release" ^
                /nologo

              echo __EXITCODE__=!ERRORLEVEL!
            """).trim()

            def exitLine = log.readLines().reverse().find { it.startsWith('__EXITCODE__=') }
            def exitCode = (exitLine?.split('=')?.last() ?: '1') as Integer

            if (exitCode == 0) {
              echo "Build APP OK"
              break
            }

            // tenta extrair Unit faltando:  error F2613: Unit 'XXXXX' not found.
            def missing = null
            for (def line : log.readLines()) {
              def m = (line =~ /error\s+F2613:\s+Unit\s+'([^']+)'\s+not\s+found/i)
              if (m.find()) { missing = m.group(1); break }
            }

            if (!missing) {
              error("Falhou, mas nao consegui identificar Unit faltando no log.\n\nLOG:\n${log}")
            }

            echo "Unit faltando detectada: ${missing}"

            // procura MissingUnit.pas nas roots
            def foundDir = null
            for (def r : roots) {
              def out = bat(returnStdout: true, script: """
                @echo off
                for /f "delims=" %%F in ('dir /b /s "${r}\\${missing}.pas" 2^>nul') do (
                  echo FOUND=%%~dpF
                  exit /b 0
                )
                echo FOUND=
              """).trim()

              def f = out.readLines().find { it.startsWith('FOUND=') }?.substring('FOUND='.length())?.trim()
              if (f) { foundDir = f.endsWith("\\") ? f[0..-2] : f; break }
            }

            if (!foundDir) {
              error("Nao achei ${missing}.pas nas roots configuradas.\nRoots:\n- " + roots.join('\n- ') + "\n\nLOG:\n${log}")
            }

            echo "Achei ${missing}.pas em: ${foundDir}"
            extraPaths.add(foundDir)

            if (i == maxTries) {
              error("Estourou limite de tentativas. Ultimo log:\n${log}")
            }
          }

          // exporta os paths resolvidos pro resto do pipeline (TESTS reaproveitar)
          env.EXTRA_UNIT_PATHS = extraPaths.toList().join(';')
          echo "EXTRA_UNIT_PATHS final: ${env.EXTRA_UNIT_PATHS}"
        }
      }
    }

    stage('Build TESTS (CalcTeste) - auto inject Units') {
      steps {
        script {
          def roots = [
            env.WEBCHARTS_DIR,
            "${env.ACBR_FONTS}\\Terceiros",
            "${env.ACBR_FONTS}",
            "%WORKSPACE%\\CalcProject"
          ]

          def extraPaths = [] as Set
          // já aproveita o que foi descoberto no APP
          if (env.EXTRA_UNIT_PATHS?.trim()) {
            env.EXTRA_UNIT_PATHS.split(';').each { p ->
              if (p?.trim()) extraPaths.add(p.trim())
            }
          }

          def maxTries = 10
          for (int i = 1; i <= maxTries; i++) {
            def appSrc = "${env.WORKSPACE}\\CalcProject"
            def unitSearch = ([appSrc, env.WEBCHARTS_DIR, env.ACBR_BASE_PATH] + extraPaths.toList()).join(';')
            unitSearch = unitSearch.replaceAll(/;{2,}/, ';')

            echo "Tentativa ${i}/${maxTries} - UnitSearchPath TESTS: ${unitSearch}"

            def log = bat(returnStdout: true, script: """
              @echo off
              setlocal
              call "${env.DELPHI_HOME}\\bin\\rsvars.bat" >nul

              cd /d "%WORKSPACE%\\CalcTeste"

              msbuild Project1.dproj /t:Build ^
                /p:Config=Release /p:Platform=Win32 ^
                /p:DCC_UnitSearchPath="${unitSearch}" ^
                /p:DCC_IncludePath="${unitSearch}" ^
                /p:DCC_OutputDir="Win32\\Release" ^
                /nologo

              echo __EXITCODE__=!ERRORLEVEL!
            """).trim()

            def exitLine = log.readLines().reverse().find { it.startsWith('__EXITCODE__=') }
            def exitCode = (exitLine?.split('=')?.last() ?: '1') as Integer

            if (exitCode == 0) {
              echo "Build TESTS OK"
              break
            }

            def missing = null
            for (def line : log.readLines()) {
              def m = (line =~ /error\s+F2613:\s+Unit\s+'([^']+)'\s+not\s+found/i)
              if (m.find()) { missing = m.group(1); break }
            }

            if (!missing) {
              error("Falhou, mas nao consegui identificar Unit faltando no log.\n\nLOG:\n${log}")
            }

            echo "Unit faltando detectada (TESTS): ${missing}"

            def foundDir = null
            for (def r : roots) {
              def out = bat(returnStdout: true, script: """
                @echo off
                for /f "delims=" %%F in ('dir /b /s "${r}\\${missing}.pas" 2^>nul') do (
                  echo FOUND=%%~dpF
                  exit /b 0
                )
                echo FOUND=
              """).trim()

              def f = out.readLines().find { it.startsWith('FOUND=') }?.substring('FOUND='.length())?.trim()
              if (f) { foundDir = f.endsWith("\\") ? f[0..-2] : f; break }
            }

            if (!foundDir) {
              error("Nao achei ${missing}.pas nas roots (TESTS).\nRoots:\n- " + roots.join('\n- ') + "\n\nLOG:\n${log}")
            }

            echo "Achei ${missing}.pas em (TESTS): ${foundDir}"
            extraPaths.add(foundDir)

            if (i == maxTries) {
              error("Estourou limite de tentativas (TESTS). Ultimo log:\n${log}")
            }
          }
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
