---
layout: post
title:  "GradleによるJavaプロジェクトのビルド"
tags: "java, gradle"
---

JavaプロジェクトのビルドツールであるGradleの使用方法の確認として、階層型マルチプロジェクト構成を持ち、ビルドにより各モジュール起動用のスクリプトファイルが作成されるというものを作成してみることにした。IDEとしてはEclipseを使用する。

- [Eclipse用プロジェクトの作成](#anchor1)
- [Eclipse上での開発](#anchor2)
- [ビルド処理の記述](#anchor3)
- [ビルド実行](#anchor4)

#### 実行環境

- OS: Mac OS X 10.11.4 x86_64
- Java: 1.8.0_77
- Gradle: 2.4.4
- Eclipse: Mars(4.5.2)

#### プロジェクト構成

```
gradle-multi-example
+--- module-core
+--- module-core-export (module-coreに依存)
+--- module-core-import (module-coreに依存)
+--- module-export-foo (実行可能。module-core-exportに依存)
\--- module-import-bar (実行可能。module-core-importに依存)

(gradle-multi-exampleがrootProjectとなる)
(gradle-multi-example以外のProjectはsubprojectsとなる)
```

<a id="anchor1"></a>

### Eclipse用プロジェクトの作成

プロジェクトの作成開始からEclipseへのプロジェクトインポートまでを行う。

#### 1. Gradleによるプロジェクト初期化

プロジェクト用に新規ディレクトリを作成し、以下のコマンドを実行する。

```bash
$ gradle init
```

#### 2. settings.gradleの編集

マルチプロジェクト構成の場合、`settings.gradle`でサブプロジェクトを定義する必要がある。1.で生成された`settings.gradle`を以下のように編集する。(ちなみに役割に対してファイル名が漠然としてる気がするけど、他にも使用用途があるのか？)

```
rootProject.name = 'gradle-multi-example'
include 'module-core'
include 'module-core-export'
include 'module-core-import'
include 'module-export-foo'
include 'module-import-bar'
```

#### 3. build.gradleの編集

Gradleによるビルド処理は基本`build.gradle`ファイルに書く。`build.gradle`の配置場所は変更もできるし、各サブプロジェクト内に分散させて配置もできるが、基本はrootProjectの直下に配置されるものを使えばオッケー。  
まずはEclipse用のリソース(.classpath等)を持ったプロジェクト作成のために必要な情報だけを記述。実際のビルドに関する処理は後で付け加えることにする。

```groovy
// 全subprojectsに共通の設定
subprojects {
    // Eclipse用のプロジェクト作成のためのplugin
    apply plugin: 'eclipse'
    apply plugin: 'java'

    // 全ソースがUTF-8であることを指定する、
    // この記述であればsourceSetが増えても対応可能。
    def defaultEncoding = 'UTF-8'
    tasks.withType(JavaCompile) {
        options.encoding = defaultEncoding
    }

    // コンパイル対象バージョンの指定
    sourceCompatibility = 1.8
    targetCompatibility = 1.8

    // version指定
    version = '0.0.1-SNAPSHOT'

    repositories {
        mavenCentral()
    }

    // 全てのprojectで設定される依存性
    dependencies {
        compile 'org.projectlombok:lombok:1.16.8'
        testCompile 'junit:junit:4.12'
    }
}

// 各subprojectに固有の設定
project(':module-core') {
}

project(':module-core-export') {
    // 個々のprojectでのみ必要な依存性
    // 他のprojectの依存性も設定可能であり、eclipseの.classpathファイルにも反映される。
    dependencies {
        compile project(':module-core')
    }
}

project(':module-core-import') {
    dependencies {
        compile project(':module-core')
    }
}

project(':module-export-foo') {
    dependencies {
        compile project(':module-core-export')
        compile 'org.apache.commons:commons-lang3:3.4'
    }
}

project(':module-import-bar') {
    dependencies {
        compile project(':module-core-import')
    }
}
```

#### 4. eclipseタスク実行

ルートプロジェクト直下で`gradle eclipse`実行。各サブプロジェクトにEclipse用リソースが作成される。

```bash
$ gradle eclipse
:module-core:eclipseClasspath
:module-core:eclipseJdt
:module-core:eclipseProject
:module-core:eclipse
.
.
.
:module-import-bar:eclipseClasspath
:module-import-bar:eclipseJdt
:module-import-bar:eclipseProject
:module-import-bar:eclipse

BUILD SUCCESSFUL
```

Eclipseから`[Import] -> Existing Projects into Workspace`により各サブプロジェクトをインポート。ライブラリ、プロジェクト間の依存関係やJavaのコンパイル対象バージョンが上記の設定通りになっていることを確認する。(commit: b0ee551)

<a id="anchor2"></a>

### Eclipse上での開発

通常通りコード補完におんぶにだっこでコーディング。以下開発中の実行方法に関してメモ。(commit: 31e82e7)

- ソースコードやリソースはsource folderとしてMavenの規約通りに作っておくと、Eclipse上からもGradleからも実行しやすい。
  - src/main/java
  - src/main/resources
  - src/test/java
  - src/test/resources
- Eclipseの場合、単純にRunで実行できるはず。
- Gradleから実行する際は、`build.gradle`の実行したいsubprojectの`project`ブロック内に以下の記述を足し、`gradle <target module>:run`のようにして実行する(e.g. `gradle module-import-bar:run`)。
  - `apply plugin: application`
  - `mainClassName = <mainClassName>`

<a id="anchor3"></a>

### ビルド処理の記述

各プロジェクトでの開発が終了したとし、`build.gradle`にビルド処理を記述していく。

#### 1. ビルド用リソースの準備

ルートプロジェクト直下にビルド成果物雛形として`base`ディレクトリを作成する。今回はビルド時にはこのディレクトリをルートプロジェクトの`build/example-gradle`ディレクトリとしてコピーし、ビルド生成物とする。  
コピー元、コピー先のディクレトリはルートプロジェクト直下に配置した`gradle.properties`から取得している。この際、プロパティ名に`.`がつくものをビルドスクリプト内で参照するには`"${foo.bar}"`ではなく`rootProject.properties['foo.bar']`のように記述する必要がある(前者だとprojectのプロパティを見に行ってしまい？、存在しないと言われる)。  
コピーにはGradleの用意するcopyタスク or メソッドを使用するととてもラク。from, into, include, excludeを指定することでかなり柔軟にコピーが可能。

```groovy
// build先のdirectory作成
// gradle.propertiesで定義しているdot表記のproperties取得は要注意
def scriptDir = rootProject.properties['build.dir']
def baseDir = rootProject.properties['base.dir']
copy {
    from "${baseDir}"
    into "${scriptDir}"
    exclude '**/*.bat'
    exclude '**/*.sh'
    includeEmptyDirs = true
}
```

#### 2. 依存ライブラリのコピー

依存ライブラリ、および各モジュールの成果物を全て`build/example-gradle/lib`下にコピーする。  
各モジュールの依存ライブラリは`subprojects`プロパティでアクセスされる各モジュールの`project.configurations.compile`プロパティを参照することで取得している。各モジュールの成果物は`project.jar.archivePath`により取得している(なお、依存ライブラリも成果物への参照も`java.io.File`オブジェクトとして取得されるので、そのままGradleの用意するcopyタスク、メソッドに渡すことができる)。

```groovy
// 依存libraryと各modleのcopy
def libDir = rootProject.properties['build.lib.dir']
def deps = []
subprojects.each { p ->
    p.configurations.compile.each { dep ->
        if (!deps.contains(dep)) {
            deps << dep
        }
    }
    def archiveJarPath = p.jar.archivePath
    if (!deps.contains(archiveJarPath)) {
        deps << archiveJarPath
    }
}
copy {
    from files(deps)
    into libDir
}
```

### 3. 実行用スクリプトファイル作成

`application`プラグインを適用したプロジェクトには、自動的に`startScripts`タスクが追加されビルド時に実行用スクリプトファイルが作成されるようになる。目的に合えば自動生成されたこのファイルを使用するのが良いが、今回は別途自分で作成することにした。copyメソッドによりあらかじめ用意したテンプレートをコピーし、同時にexpandメソッドによりパラメータの展開も行っている。

```groovy
// 各moduleの実行用スクリプトファイルの作成
def baseBinDir = rootProject.properties['base.bin.dir']
def binDir = rootProject.properties['build.bin.dir']
def execLibDir = rootProject.properties['exec.lib.dir']
subprojects.each { p ->
    if (p.plugins.hasPlugin('application')) {
        def classPath = [p.jar.archivePath] + p.configurations.compile
        copy {
            from baseBinDir
            into binDir
            include "**/${p.ext.scriptType}.sh"
            rename '.*\\.sh', p.name + '.sh'
            expand(
                mainClassName: p.mainClassName,
                classPath: classPath.collect{x -> "${execLibDir}/${x.name}"}.join(':')
            )
        }
    }
}
```

課題としては

- 異なるOSへのスッキリした対応。copyメソッドを2回実行するしか無いんかな...
- 本来は`startScripts`タスクをoverwriteしたかったのだがうまくいかず。`gradle module-import-bar:startScripts`のようにタスク単体で起動するとoverwriteできていることを確認できるが、`gradle build`のようにビルド時に実行される流れではoverwriteされず。原因がわからず諦めた。

<a id="anchor4"></a>

### ビルド実行

ビルドにより作成されたバッチを実行することでプログラムを起動できることを確認した。(commit: 96c4d99)

```bash
$ gradle clean build buildScript
$ export HOME_DIR=./build/example-gradle
$ ./build/example-gradle/bin/module-import-bar.sh

Execute import batch
Hello, import
```

### 使ってみた印象

- Mavenに比べるとだいぶ書きやすい。
- API document等資料も充実している。
- Gradleへの理解が完璧ではなくても、Projectの各種情報へのアクセス法、Gradleの用意するファイル操作API、簡単なGroovyの知識があればある程度のことはできそう。

### サンプル用リポジトリ

- [gradle-multi-example](https://github.com/tiqwab/example/tree/master/gradle-multi-example)
