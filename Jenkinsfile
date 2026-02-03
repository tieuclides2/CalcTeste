pipeline {
  agent any
  options { timestamps() }

  parameters {
    booleanParam(name: 'CLEAN_GLOBAL_CACHE', defaultValue: false, description: 'Limpa o cache global de DCU (para quando atualizar vendors como ACBr).')
    choice(name: 'PLATFORM', choices: ['Win32', 'Win64'], description: 'Plataforma do build.')
    choice(name: 'CONFIG', choices: ['Release', 'Debug'], description: 'Configuração do build.')
  }

  environment {
    DELPHI_HOME = 'C:\\DelphiCompiler\\23.0'

    // Cache global centralizado
    CACHE_ROOT = 'C:\\Jenkins\\delphi-qa\\JenkinsCache'

    // Vendors (padrão: tudo dentro de C:\DelphiCompiler\Componentes)
    COMPONENTES_ROOT = 'C:\\DelphiCompiler\\Componentes'
    WEBCHARTS_DIR    = 'C:\\DelphiCompiler\\Componentes\\TBGWebCharts'
    ACBR_DIR         = 'C:\\DelphiCompiler\\Componentes\\ACBr'
    BCEDITOR_DIR     = 'C:\\DelphiCompiler\\Componentes\\BCEditor'
    REDSIS_DIR       = 'C:\\DelphiCompiler\\Componentes\\RedsisComponents'
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
        bat """
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
          if not exist "%COMPONENTES_ROOT%" (
            echo ERRO: Pasta Componentes nao existe: %COMPONENTES_ROOT%
            exit /b 1
          )
          dir "%COMPONENTES_ROOT%" /ad
          echo.

          echo === Cache Root ===
          if not exist "%CACHE_ROOT%" (
            mkdir "%CACHE_ROOT%"
          )
          dir "%CACHE_ROOT%" /ad
        """
      }
    }

    stage('Resolver paths (ACBr + vendors)') {
      steps {
        script {
          // ACBr paths mínimos + Terceiros (onde geralmente ficam synapse/faststring/gziputils etc)
          // (Como você já confirmou que seus terceiros ficam dentro da estrutura do ACBr, isso cobre bem.)
          env.ACBR_PATH = [
            "${env.ACBR_DIR}\\Fontes\\ACBrComum",
            "${env.ACBR_DIR}\\Fontes\\ACBrDiversos",
            "${env.ACBR_DIR}\\Fontes\\ACBrTCP",
            "${env.ACBR_DIR}\\Fontes\\Terceiros",
            "${env.ACBR_DIR}\\Fontes\\Terceiros\\synalist",
            "${env.ACBR_DIR}\\Fontes\\Terceiros\\FastStringReplace",
            "${env.ACBR_DIR}\\Fontes\\Terceiros\\GZIPUtils"
          ].join(';')

          // Cache DCU global por projeto (evita “entupir” Win32\Release do workspace)
          env.DCU_CACHE_APP   = "${env.CACHE_ROOT}\\DCU\\D23\\${params.PLATFORM}\\${params.CONFIG}\\CalcProject"
          env.DCU_CACHE_TESTS = "${env.CACHE_ROOT}\\DCU\\D23\\${params.PLATFORM}\\${params.CONFIG}\\CalcTeste"

          // UnitSearchPath (cache primeiro ajuda a “reusar”, depois fontes)
          // Obs: se não achar DCU no cache, ele cai para fontes e recompila, jogando o DCU no cache (via DCC_DcuOutput)
          env.UNIT_PATH_APP = [
            env.DCU_CACHE_APP,
            env.WEBCHARTS_DIR,
            env.ACBR_PATH,
            env.BCEDITOR_DIR,
            env.REDSIS_DIR
          ].join(';')

          env.UNIT_PATH_TESTS = [
            env.DCU_CACHE_TESTS,
            // cache do app ajuda o projeto de testes a achar units do app sem recompilar
            env.DCU_CACHE_APP,
            "${env.WORKSPACE}\\CalcProject",
            env.WEBCHARTS_DIR,
            env.ACBR_PATH,
            env.BCEDITOR_DIR,
            env.REDSIS_DIR
          ].join(';')

          echo "WEBCHARTS_DIR: ${env.WEBCHARTS_DIR}"
          echo "ACBR_DIR: ${env.ACBR_DIR}"
          echo "ACBR_PATH: ${env.ACBR_PATH}"
          echo "DCU_CACHE_APP: ${env.DCU_CACHE_APP}"
          echo "DCU_CACHE_TESTS: ${env.DCU_CACHE_TESTS}"
          echo "UNIT_PATH_APP: ${env.UNIT_PATH_APP}"
          echo "UNIT_PATH_TESTS: ${env.UNIT_PATH_TESTS}"
        }

        bat """
          @echo on
          if not exist "%WEBCHARTS_DIR%\\View.WebCharts.pas" (
            echo ERRO: View.WebCharts.pas nao encontrado em %WEBCHARTS_DIR%
            exit /b 1
          )

          if not exist "%ACBR_DIR%\\Fontes" (
            echo ERRO: ACBr Fontes nao encontrado em %ACBR_DIR%\\Fontes
            exit /b 1
          )

          if not exist "%BCEDITOR_DIR%" (
            echo AVISO: BCEditor nao encontrado em %BCEDITOR_DIR%
          )

          if not exist "%REDSIS_DIR%" (
            echo AVISO: RedsisComponents nao encontrado em %REDSIS_DIR%
          )
        """
      }
    }

    stage('Limpar cache global (opcional)') {
      when { expression { return params.CLEAN_GLOBAL_CACHE } }
      steps {
        bat """
          @echo on
          echo Limpando cache global...
          if exist "%DCU_CACHE_APP%"   rmdir /s /q "%DCU_CACHE_APP%"
          if exist "%DCU_CACHE_TESTS%" rmdir /s /q "%DCU_CACHE_TESTS%"
          echo OK.
        """
      }
    }

    stage('Build APP (CalcProject) usando cache global DCU') {
      steps {
        dir('CalcProject') {
          bat """
            @echo on
            call "%DELPHI_HOME%\\bin\\rsvars.bat"

            if not exist "%DCU_CACHE_APP%" mkdir "%DCU_CACHE_APP%"

            echo === MSBUILD Calc.dproj CFG=${params.CONFIG} PLAT=${params.PLATFORM} ===
            echo DCU_CACHE_APP=%DCU_CACHE_APP%
            echo UNIT_PATH_APP=%UNIT_PATH_APP%

            msbuild "Calc.dproj" /t:Build ^
              /p:Config=${params.CONFIG} /p:Platform=${params.PLATFORM} ^
              /p:DCC_UnitSearchPath="%UNIT_PATH_APP%" ^
              /p:DCC_IncludePath="%UNIT_PATH_APP%" ^
              /p:DCC_DcuOutput="%DCU_CACHE_APP%" ^
              /p:DCC_OutputDir="${params.PLATFORM}\\${params.CONFIG}"

            if errorlevel 1 exit /b 1
          """
        }
      }
    }

    stage('Build TESTS (CalcTeste) usando cache global DCU') {
      steps {
        dir('CalcTeste') {
          bat """
            @echo on
            call "%DELPHI_HOME%\\bin\\rsvars.bat"

            if not exist "%DCU_CACHE_TESTS%" mkdir "%DCU_CACHE_TESTS%"

            echo === MSBUILD Project1.dproj CFG=${params.CONFIG} PLAT=${params.PLATFORM} ===
            echo DCU_CACHE_TESTS=%DCU_CACHE_TESTS%
            echo UNIT_PATH_TESTS=%UNIT_PATH_TESTS%

            msbuild "Project1.dproj" /t:Build ^
              /p:Config=${params.CONFIG} /p:Platform=${params.PLATFORM} ^
              /p:DCC_UnitSearchPath="%UNIT_PATH_TESTS%" ^
              /p:DCC_IncludePath="%UNIT_PATH_TESTS%" ^
              /p:DCC_DcuOutput="%DCU_CACHE_TESTS%" ^
              /p:DCC_OutputDir="${params.PLATFORM}\\${params.CONFIG}"

            if errorlevel 1 exit /b 1
          """
        }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        dir('CalcTeste') {
          bat """
            @echo on
            set "TEST_EXE=%WORKSPACE%\\CalcTeste\\${params.PLATFORM}\\${params.CONFIG}\\Project1.exe"

            if not exist "%TEST_EXE%" (
              echo ERRO: Test EXE nao encontrado: %TEST_EXE%
              dir "%WORKSPACE%\\CalcTeste\\${params.PLATFORM}\\${params.CONFIG}"
              exit /b 1
            )

            "%TEST_EXE%"
            exit /b %ERRORLEVEL%
          """
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts allowEmptyArchive: true,
        artifacts: 'CalcProject\\Win32\\**\\*.exe, CalcProject\\Win64\\**\\*.exe, CalcProject\\**\\*.dll, CalcProject\\**\\*.map',
        fingerprint: true

      archiveArtifacts allowEmptyArchive: true,
        artifacts: 'CalcTeste\\Win32\\**\\*.exe, CalcTeste\\Win64\\**\\*.exe, CalcTeste\\**\\*.xml, CalcTeste\\**\\*.log',
        fingerprint: true
    }
  }
}
