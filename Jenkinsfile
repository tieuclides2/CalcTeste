pipeline {
  agent { label 'delphi-qa' }

  environment {
    // Delphi
    RSVARS = 'C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat'

    // Vendors base
    COMP_BASE = 'C:\\DelphiCompiler\\Componentes'

    // Cache global
    JENKINS_CACHE = 'C:\\Jenkins\\delphi-qa\\JenkinsCache'

    // defaults
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
        bat '''@echo off
        echo === WHOAMI / CONTEXTO ===
        whoami
        echo.

        echo === Delphi Env ===
        if not exist "%RSVARS%" (
          echo ERRO: rsvars.bat nao encontrado em %RSVARS%
          exit /b 1
        )
        call "%RSVARS%"
        where msbuild
        where dcc32
        echo.

        echo === Componentes ===
        if not exist "%COMP_BASE%" (
          echo ERRO: Pasta Componentes nao existe: %COMP_BASE%
          exit /b 1
        )
        dir "%COMP_BASE%" /ad
        echo.

        echo === Cache ===
        if not exist "%JENKINS_CACHE%" mkdir "%JENKINS_CACHE%"
        if not exist "%JENKINS_CACHE%\\DCU" mkdir "%JENKINS_CACHE%\\DCU"
        dir "%JENKINS_CACHE%" /ad
        '''
      }
    }

    stage('Resolver roots (vendors + FastReport)') {
      steps {
        script {
          def webcharts = "${env.COMP_BASE}\\TBGWebCharts"
          def acbr      = "${env.COMP_BASE}\\ACBr"
          def bceditor  = "${env.COMP_BASE}\\BCEditor"
          def redsis    = "${env.COMP_BASE}\\RedsisComponents"

          def acbrPath = [
            "${acbr}\\Fontes\\ACBrComum",
            "${acbr}\\Fontes\\ACBrDiversos",
            "${acbr}\\Fontes\\ACBrTCP",
            "${acbr}\\Fontes\\Terceiros\\synalist",
            "${acbr}\\Fontes\\Terceiros\\FastStringReplace",
            "${acbr}\\Fontes\\Terceiros\\GZIPUtils",
            "${acbr}\\Fontes\\Terceiros"
          ].join(';')

          def frBase    = "${env.COMP_BASE}\\Fast Reports\\VCL\\2025.2.2"
          def frSources = "${frBase}\\Sources"

          def frDcuCandidates = [
            "${frBase}\\LibRS29\\VCL\\Win32",
            "${frBase}\\LibRS29\\VCL\\Win32x",
            "${frBase}\\Sources\\LibRS29\\VCL\\Win32",
            "${frBase}\\Sources\\LibRS29\\VCL\\Win32x"
          ]

          def frDcus = frDcuCandidates.findAll { p -> fileExists(p) }
          echo "FastReport DCU dirs detectados: ${frDcus}"

          env.UNIT_PATH = ([
            webcharts,
            acbrPath
          ] + frDcus + [
            frSources
          ]).join(';')

          echo "UNIT_PATH: ${env.UNIT_PATH}"
        }
      }
    }

    stage('Build APP (CalcProject)') {
      steps {
        dir('CalcProject') {
          bat """@echo off
          call "%RSVARS%"
          echo === MSBUILD Calc.dproj CFG=%CFG% PLAT=%PLAT% ===

          set "DCU_OUT=%JENKINS_CACHE%\\DCU\\CalcProject\\%PLAT%\\%CFG%"
          if not exist "%DCU_OUT%" mkdir "%DCU_OUT%"

          msbuild "Calc.dproj" /t:Build ^
            /p:Config=%CFG% /p:Platform=%PLAT% ^
            /p:DCC_UnitSearchPath="${env.UNIT_PATH}" ^
            /p:DCC_IncludePath="${env.UNIT_PATH}" ^
            /p:DCC_DcuOutput="%DCU_OUT%" ^
            /p:DCC_OutputDir="Win32\\Release" ^
            /fl /flp:logfile=msbuild_calc.log;verbosity=minimal

          exit /b %ERRORLEVEL%
          """
        }
      }
    }

    stage('Build TESTS (CalcTeste)') {
      steps {
        dir('CalcTeste') {
          bat """@echo off
          call "%RSVARS%"
          echo === MSBUILD Project1.dproj (TESTS) CFG=%CFG% PLAT=%PLAT% ===

          if not exist "Project1.dproj" (
            echo ERRO: Project1.dproj nao encontrado em %CD%
            dir
            exit /b 2
          )

          set "DCU_OUT=%JENKINS_CACHE%\\DCU\\CalcTeste\\%PLAT%\\%CFG%"
          if not exist "%DCU_OUT%" mkdir "%DCU_OUT%"

          msbuild "Project1.dproj" /t:Build ^
            /p:Config=%CFG% /p:Platform=%PLAT% ^
            /p:DCC_UnitSearchPath="${env.UNIT_PATH}" ^
            /p:DCC_IncludePath="${env.UNIT_PATH}" ^
            /p:DCC_DcuOutput="%DCU_OUT%" ^
            /p:DCC_OutputDir="Win32\\Release" ^
            /fl /flp:logfile=msbuild_tests.log;verbosity=minimal

          exit /b %ERRORLEVEL%
          """
        }
      }
    }

    stage('Run unit tests (DUnitX)') {
      steps {
        dir('CalcTeste') {
          bat """@echo off
          echo === RUN TESTS ===

          if not exist "Win32\\Release\\Project1.exe" (
            echo ERRO: Test runner nao encontrado: Win32\\Release\\Project1.exe
            dir "Win32\\Release"
            exit /b 3
          )

          rem DUnitX console runner costuma aceitar parametros; se nao aceitar, ele roda e retorna errorlevel.
          "Win32\\Release\\Project1.exe"
          set ERR=%ERRORLEVEL%
          echo Test runner exit code: %ERR%

          exit /b %ERR%
          """
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/*.log, **/Win32/Release/*.exe', allowEmptyArchive: true
    }
  }
}
