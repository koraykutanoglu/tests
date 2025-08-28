---

## Nedir?

Helm Chart Unit Test, Kubernetes üzerinde çalışan uygulamalar için hazırlanan Helm chart’larının doğru şekilde çalışıp çalışmadığını küçük, hızlı ve otomatik testlerle doğrulamayı sağlar. **helm-unittest** eklentisi kullanılarak yazılan bu testler, chart’ın `values.yaml` parametreleri farklı değerler aldığında oluşturulan Kubernetes manifestlerinin beklendiği gibi üretilip üretilmediğini kontrol eder.

## Gereksinimler

- [Helm Kurulumu (v2 & v3)](https://www.notion.so/Helm-Kurulumu-v2-v3-04a8b7de453e41cf81fced1ddd92e5f0?pvs=21)

## Helm Unit Test Plugin

Öncelikle helm unit test plugin kurulumu için aşağıdaki komutu çalıştırabilirsiniz.

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest.git
```

İlgili plugin kurulumunu bu şekilde test edebilirsiniz:

```bash
koraykutanoglu@Koray-Mac % helm unittest
Error: requires at least 1 arg(s), only received 0
Usage:
  unittest [flags] CHART [...]

Flags:
      --chart-tests-path string   chart-tests-path the folder location relative to the chart where a helm chart to render test suites is located
      --color                     enforce printing colored output even stdout is not a tty. Set to false to disable color
  -d, --debugPlugin               enable verbose output
  -q, --failfast                  actually directly quit testing, when a test is failed
  -f, --file stringArray          glob paths of test files location, default to tests/*_test.yaml
  -h, --help                      help for unittest
  -o, --output-file string        output-file the file where testresults are written in JUnit format, defaults no output is written to file
  -t, --output-type string        output-type the file-format where testresults are written in, accepted types are (JUnit, NUnit, XUnit, Sonar) (default "XUnit")
      --strict                    strict parse the testsuites
  -u, --update-snapshot           update the snapshot cached if needed, make sure you review the change before update
  -v, --values stringArray        absolute or glob paths of values files location to override helmchart values
  -s, --with-subchart charts      include tests of the subcharts within charts folder (default true)

requires at least 1 arg(s), only received 0
```

## Test Yapısının Oluşturulması

Unit Test yazabilmek için öncelikle chart yapımızın oluşturulması gerekiyor. Burada test dosyaları `tests/` dizini altında yer alıyor. Buradaki dosyaların test dosyası olarak değerlendirilebilmesi için _test.yaml ekiyle bitmesi gerekiyor.

```yaml
helm-chart/
  Chart.yaml
  values.yaml
  templates/
  tests/
    my_test.yaml
```

Chart yapısını oluşturabilmeniz için en temel seviye dosyalar aşağıdaki gibidir:

                           `Chart.yaml`                                                                     `values.yaml`

```yaml
apiVersion: v2
name: helm-chart
description: demo
type: application
version: 0.1.0
appVersion: "1.0"
```

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: "latest"
  pullPolicy: IfNotPresent
```

`templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

`tests/my_test.yaml`

```yaml
suite: Basic deployment tests
templates:
  - templates/deployment.yaml

tests:
  - it: should set replicas to 3 when replicaCount=3
    set:
      replicaCount: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3

  - it: should use nginx:latest as default image
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: "nginx:latest"
```

burada yapılan test `templates/deployment.yaml` template dosyası içerisindeki replica ve image değeri üzerinedir. Eğer values.yaml’dan ezilmediyse ve replica 3, image ise nginx:latest değil ise unit test hata ile sonuçlanacaktır.

## Temel Test Senaryoların Çalıştırılması

Aşağıdaki komut ile test’i başlatabilirsiniz.

```yaml
helm unittest .
```

Eğer sonuç başarılıysa aşağıdaki gibi çıktı verecektir.

```yaml
koraykutanoglu@Koray-Mac Chart-Test % helm unittest .

### Chart [ helm-chart ] .

 PASS  Basic deployment tests   tests/my_test.yaml

Charts:      1 passed, 1 total
Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshot:    0 passed, 0 total
Time:        2.377ms
```

Diyelim ki deployment yaml’da bir çalışma yaptım ve bu çalışma sonucunda replicas değerim kendi değerini values.yaml’dan almak yerine artık statik olarak 1 gelmeye başladı. Yani bu değer bizim koşul olarak sunduğumuz 3 değerinden farklı. Bu durumda tekrar test çalıştırdığımızda testimiz hatalı bitecektir. Çünkü bu değer values.yaml’dan gelmiyor ve ayarlamış olduğumuz default değeri vermiyor.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

Testi çalıştırın:

```yaml
helm unittest .
```

Test çalıştırıldığında hatalı bitip neden hatalı bittiği açıklamada gösterilecektir.

```bash
koraykutanoglu@Koray-Mac Chart-Test % helm unittest .

### Chart [ helm-chart ] .

 FAIL  Basic deployment tests   tests/my_test.yaml
        - should set replicas to 3 when replicaCount=3

                - asserts[0] `equal` fail
                        Template:       helm-chart/templates/deployment.yaml
                        DocumentIndex:  0
                        ValuesIndex:    0
                        Path:   spec.replicas
                        Expected to equal:
                                3
                        Actual:
                                1
                        Diff:
                                --- Expected
                                +++ Actual
                                @@ -1,2 +1,2 @@
                                -3
                                +1

Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 1 passed, 2 total
Snapshot:    0 passed, 0 total
Time:        3.463583ms

Error: plugin "unittest" exited with error
```

Test dosyamızda farklı senaryolar olabilir. Örneğin ben replicas değerimin her şekilde 3 gelmesini isteyebilirim. Values.yaml’da ayarlansa bile. Bu durumda my_test.yaml dosyanızın içeriğini aşağıdaki gibi ayarlamanız gerekir.

```bash
suite: Always 3 replicas
templates:
  - templates/deployment.yaml

tests:
  - it: should always have replicas set to 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3
```

Ardından values.yaml içerisinde replicas değerinizi 1 yapın.

```bash
replicaCount: 1
```

Unit test çalıştırın:

```bash
helm unittest .
```

Hata alırsınız çünkü siz values.yaml üzerinde ayarlansa dahi replicas değerinin 3 gelmesini istiyorsunuz.

```bash
koraykutanoglu@Koray-Mac Chart-Test % helm unittest .

### Chart [ helm-chart ] .

 FAIL  Always 3 replicas        tests/my_test.yaml
        - should always have replicas set to 3

                - asserts[0] `equal` fail
                        Template:       helm-chart/templates/deployment.yaml
                        DocumentIndex:  0
                        ValuesIndex:    0
                        Path:   spec.replicas
                        Expected to equal:
                                3
                        Actual:
                                2
                        Diff:
                                --- Expected
                                +++ Actual
                                @@ -1,2 +1,2 @@
                                -3
                                +2

Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 0 passed, 1 total
Snapshot:    0 passed, 0 total
Time:        1.092042ms

Error: plugin "unittest" exited with error
```

## Parametre Override (set)

`set` özelliği, test sırasında **values.yaml’daki parametreleri geçici olarak değiştirmeyi** sağlar. Böylece farklı kombinasyonları tek test içinde deneyebilirsin.

- Bu örnekte replicaCount değerimi test içerisinde 5 olarak set ediyorum ve 5 olarak gelmesini bekliyorum. Bu ayarladığım parametre sonucunda elde etmek istediğim değeri verip vermediğini kontrol ediyor.

```bash
suite: Deployment Tests
templates:
  - templates/deployment.yaml

tests:
  - it: should set replicas to 5
    set:
      replicaCount: 5
    asserts:
      - equal:
          path: spec.replicas
          value: 5
```

Values.yaml’da hangi değerin yazdığının pek bir önemi yoktur. 

```bash
replicaCount: 3
image:
  repository: nginx
  tag: latest
```

Eğer siz deployment.yaml’da replicaCount değerini replica olarak ayarlarsanız yani test parametreyi set edecek ama elde etmek istediği sonucu alamamasını sağlarsanız test hata verir.

```bash
replicas: {{ .Values.replicaCount }}
↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
replicas: {{ .Values.replica }}
```

Testi çalıştırdığınızda beklediğim değer 5 ancak null değeri aldım şekilde hata ile karşılaşırsınız.

- Bu sadece replicas değeri için değil, test için bir feature parametre oluşturduğunuzda veya geliştirdiğinizde o feature parametreyi set etmeniz durumunda çıktı olarak beklediğiniz değerleri test olarak yazabilmeniz anlamına da gelir.

```bash
koraykutanoglu@Koray-Mac Chart-Test % helm unittest .

### Chart [ helm-chart ] .

 FAIL  Deployment Tests tests/my_test.yaml
        - should set replicas to 5

                - asserts[0] `equal` fail
                        Template:       helm-chart/templates/deployment.yaml
                        DocumentIndex:  0
                        ValuesIndex:    0
                        Path:   spec.replicas
                        Expected to equal:
                                5
                        Actual:
                                null
                        Diff:
                                --- Expected
                                +++ Actual
                                @@ -1,2 +1,2 @@
                                -5
                                +null

Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 0 passed, 1 total
Snapshot:    0 passed, 0 total
Time:        4.33ms
```

## Null / Non-Null Kontroller

Null / Non-Null kontroller ile bir değerin çıktısının boş gelip gelmediğini kontrol ettirebiliriz. 

- Deployment.yaml’ımızın bu şekilde olduğunu düşünün.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: null
    spec:
      containers:
        - name: demo
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Aşağıdaki test ile hem replicas hem de labels kısmındaki app değerinin boş gelip gelmemesini kontrol edebiliriz. Buradaki en temel fark replicas kısmında notEqual, labels kısmında ise equal kullandığımız için:

- replicas değeri herhangi bir şekilde boş veya null gelirse hata verir.
- labels kısmındaki app değeri herhangi bir şekilde değer alırsa hata verir.

Replicas değerinin boş gelmemesini kontrol etmem mantıklı olabilir, aynı şekilde bir parametreyi disable ettiğimde deployment’ta bir env’in gelmemesini de kontrol etmek isteyebilirim.

```yaml
suite: Deployment Tests
templates:
  - templates/deployment.yaml

tests:
  - it: replicaCount should be set
    asserts:
      - notEqual:
          path: spec.replicas
          value: null

  - it: optionalValue should be null
    asserts:
      - equal:
          path: spec.template.metadata.labels.app
          value: null
```
