@NonCPS
String extractMissingUnit(String logText) {
  if (logText == null) return null

  def p = java.util.regex.Pattern.compile(
    "error\\s+F2613:\\s+Unit\\s+'([^']+)'\\s+not\\s+found",
    java.util.regex.Pattern.CASE_INSENSITIVE
  )
  def m = p.matcher(logText)
  if (m.find()) return m.group(1)
  return null
}

pipeline {
  agent { label 'delphi-qa' }

  environment {
    // Delphi
    RSVARS = 'C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat'

    // Vendors base
    COMP_BASE = 'C:\\DelphiCompiler\\Componentes'

    // Cache global (não precisa pré-criar; o pipeline cria)
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

          // ACBr (inclui GZIPUtils e Terceiros)
          def acbrPath = [
            "${acbr}\\Fontes\\ACBrComum",
            "${acbr}\\Fontes\\ACBrDiversos",
            "${acbr}\\Fontes\\ACBrTCP",
            "${acbr}\\Fontes\\Terceiros\\synalist",
            "${acbr}\\Fontes\\Terceiros\\FastStringReplace",
            "${acbr}\\Fontes\\Terceiros\\GZIPUtils",
            "${acbr}\\Fontes\\Terceiros"
          ].join(';')

          // FastReport (ajuste a versão se mudar)
          def frBase    = "${env.COMP_BASE}\\Fast Reports\\VCL\\2025.2.2"
          def frSources = "${frBase}\\Sources"
          def frLib     = "${frBase}\\LibRS29"

          env.UNIT_PATH = [
            webcharts,
            acbrPath,
            frSources,
            frLib
          ].join(';')

          // roots para busca “auto-inject” (se você tiver essa lógica ativa)
          env.VENDOR_ROOTS = [
            webcharts,
            acbr, "${acbr}\\Fontes", "${acbr}\\Fontes\\Terceiros",
            bceditor,
            redsis,
            frBase,
            frSources,
            frLib
          ].join(';')

          echo "UNIT_PATH: ${env.UNIT_PATH}"
          echo "VENDOR_ROOTS: ${env.VENDOR_ROOTS}"
        }
      }
    }

    stage('Build APP (CalcProject)') {
      steps {
        dir('CalcProject') {
          script {
            // log file do msbuild
            def mslog = "msbuild_calc_try1.log"

            // build
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
              /p:DCC_OutputDir="Win32\\\\Release" ^
              /fl /flp:logfile=${mslog};verbosity=minimal

            exit /b %ERRORLEVEL%
            """

            // se falhar, tenta extrair a unit faltando (sem sandbox-bans)
            def txt = readFile(mslog)
            def missing = extractMissingUnit(txt)
            if (missing != null) {
              echo "Unit faltando detectada: ${missing}"
              error("Falha na compilação: unit '${missing}' não encontrada. Ajuste UNIT_PATH/VENDOR_ROOTS para incluir a pasta onde está ${missing}.pas")
            }
          }
        }
      }
    }

    // Se você quiser reativar testes depois, a gente ajusta quando você confirmar qual .dproj de testes.
    /*
    stage('Build TESTS (CalcTeste)') {
      steps {
        dir('CalcTeste') {
          bat '''@echo off
          call "%RSVARS%"
          set "DCU_OUT=%JENKINS_CACHE%\\DCU\\CalcTeste\\%PLAT%\\%CFG%"
          if not exist "%DCU_OUT%" mkdir "%DCU_OUT%"

          msbuild "SEU_TESTE.dproj" /t:Build ^
            /p:Config=%CFG% /p:Platform=%PLAT% ^
            /p:DCC_UnitSearchPath="%UNIT_PATH%" ^
            /p:DCC_IncludePath="%UNIT_PATH%" ^
            /p:DCC_DcuOutput="%DCU_OUT%" ^
            /fl /flp:logfile=msbuild_calcteste.log;verbosity=minimal
          '''
        }
      }
    }
    */
  }
}
