pipeline {
  agent { label 'delphi-qa' }

  environment {
    RSVARS = 'C:\\DelphiCompiler\\23.0\\bin\\rsvars.bat'
    COMP_BASE = 'C:\\DelphiCompiler\\Componentes'
    JENKINS_CACHE = 'C:\\Jenkins\\delphi-qa\\JenkinsCache'
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

    stage('Resolver roots (vendors + FastReport + BCEditor + Redsis)') {
      steps {
        script {
          def webcharts = "${env.COMP_BASE}\\TBGWebCharts"
          def acbr      = "${env.COMP_BASE}\\ACBr"
          def bceditor  = "${env.COMP_BASE}\\BCEditor"
          def redsis    = "${env.COMP_BASE}\\RedsisComponents"

          // Redsis (bases)
          def redsisCandidates = [
            "${redsis}\\Sources",
            "${redsis}\\Source",
            "${redsis}\\RDComponents\\Sources",
            redsis
          ]
          def redsisBase = redsisCandidates.findAll { p -> fileExists(p) }
          echo "Redsis dirs base detectados: ${redsisBase}"

          // ACBr
          def acbrPath = [
            "${acbr}\\Fontes\\ACBrComum",
            "${acbr}\\Fontes\\ACBrDiversos",
            "${acbr}\\Fontes\\ACBrTCP",
            "${acbr}\\Fontes\\Terceiros\\synalist",
            "${acbr}\\Fontes\\Terceiros\\FastStringReplace",
            "${acbr}\\Fontes\\Terceiros\\GZIPUtils",
            "${acbr}\\Fontes\\Terceiros"
          ].join(';')

          // BCEditor
          def bcCandidates = [
            "${bceditor}\\Source",
            "${bceditor}\\Sources",
            "${bceditor}\\Source\\Highlighter",
            "${bceditor}\\Sources\\Highlighter",
            "${bceditor}\\Source\\BCEditor",
            "${bceditor}\\Sources\\BCEditor"
          ]
          def bcPaths = bcCandidates.findAll { p -> fileExists(p) }
          echo "BCEditor dirs detectados: ${bcPaths}"

          // FastReport
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

          // ===== Auto-detect de units específicas (sem recursão no search path) =====
          // A ideia: encontrar o .pas e adicionar o diretório pai no UNIT_PATH.

          def extraUnitDirs = []

          def findUnitDirs = { String unitPasName ->
            // usa DIR /S /B e pega o diretório pai de cada ocorrência
            def out = bat(
              script: """@echo off
              dir /s /b "${redsis}\\${unitPasName}" 2>nul
              """,
              returnStdout: true
            ).trim()

            if (!out) {
              echo "Nao encontrado: ${unitPasName}"
              return []
            }

            def paths = out.split("\\r?\\n").collect { it.trim() }.findAll { it }
            echo "Encontrado ${unitPasName}: ${paths}"

            // extrai diretórios (remove \\arquivo.pas)
            def dirs = paths.collect { p -> p.replaceAll('\\\\[^\\\\]+$', '') }.unique()
            return dirs
          }

          // uFDCMemTable e dependência atual uRDConnection
          extraUnitDirs.addAll(findUnitDirs('uFDCMemTable.pas'))
          extraUnitDirs.addAll(findUnitDirs('uRDConnection.pas'))

          extraUnitDirs = extraUnitDirs.unique()
          echo "Dirs extras (units Redsis) detectados: ${extraUnitDirs}"

          // UNIT_PATH final
          env.UNIT_PATH = ([
            webcharts
          ] + bcPaths + redsisBase + extraUnitDirs + [
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

          set "APP_SRC=%WORKSPACE%\\CalcProject"
          set "TEST_UNIT_PATH=%APP_SRC%;${env.UNIT_PATH}"

          echo APP_SRC=%APP_SRC%
          echo TEST_UNIT_PATH=%TEST_UNIT_PATH%

          msbuild "Project1.dproj" /t:Build ^
            /p:Config=%CFG% /p:Platform=%PLAT% ^
            /p:FrameworkType=VCL ^
            /p:DCC_Namespace="Winapi;System.Win;Data.Win;Datasnap.Win;Web.Win;Soap.Win;Xml.Win;Bde;System;Xml;Data;Datasnap;Web;Soap;Vcl;Vcl.Imaging;Vcl.Touch;Vcl.Samples;Vcl.Shell" ^
            /p:DCC_UnitScopeNames="Winapi;System;Data;Datasnap;Web;Soap;Xml;Bde;Vcl" ^
            /p:DCC_UnitSearchPath="%TEST_UNIT_PATH%" ^
            /p:DCC_IncludePath="%TEST_UNIT_PATH%" ^
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
