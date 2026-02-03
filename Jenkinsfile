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

      // FastReport base
      def frBase    = "${env.COMP_BASE}\\Fast Reports\\VCL\\2025.2.2"
      def frSources = "${frBase}\\Sources"

      // DCUs do FastReport (Win32). Alguns pacotes usam Win32, outros Win32x.
      def frDcuCandidates = [
        "${frBase}\\LibRS29\\VCL\\Win32",
        "${frBase}\\LibRS29\\VCL\\Win32x",
        "${frBase}\\Sources\\LibRS29\\VCL\\Win32",
        "${frBase}\\Sources\\LibRS29\\VCL\\Win32x"
      ]

      // mantém só os diretórios que existem
      def frDcus = frDcuCandidates.findAll { p -> fileExists(p) }

      echo "FastReport DCU dirs detectados: ${frDcus}"

      // Prioridade: DCU primeiro, depois Sources
      env.UNIT_PATH = ([
        webcharts,
        acbrPath
      ] + frDcus + [
        frSources
      ]).join(';')

      env.VENDOR_ROOTS = [
        webcharts,
        acbr, "${acbr}\\Fontes", "${acbr}\\Fontes\\Terceiros",
        bceditor,
        redsis,
        frBase,
        frSources
      ].join(';')

      echo "UNIT_PATH: ${env.UNIT_PATH}"
      echo "VENDOR_ROOTS: ${env.VENDOR_ROOTS}"
    }
  }
}
