---
title: "Gradle Kotlin DSLを使ってみる (build.gradleのbuild.gradle.kts化)"
date: 2017-12-20T14:18:00+09:00
draft: false
tags: ["Kotlin", "Gradle"]
---
### はじめに
現状、Gradle Kotlin DSLなファイルを編集中に大量のLintエラーが出たり、それを解消するための再読込の仕組みが見当たらないなど、あまり積極的におすすめできる段階ではないということを予め述べておきます。

### 結論が見たい方のために
最終的に出来上がった `build.gradle.kts` 及びその周辺ファイルをまず載せておきます。

`build.gradle.kts` :
```kotlin
import org.gradle.kotlin.dsl.*
import org.jetbrains.kotlin.gradle.dsl.Coroutines

buildscript {
    repositories {
        mavenCentral()
    }
}

plugins {
    application
    kotlin("jvm") version "1.2.10"
}

application {
    mainClassName = "io.ktor.server.netty.DevelopmentEngine"
}

group = "jp.example.geckour"
version = "1.0-SNAPSHOT"

repositories {
    jcenter()
    mavenCentral()
    maven("https://dl.bintray.com/kotlin/kotlinx")
    maven("http://dl.bintray.com/kotlin/ktor")
    maven("http://dl.bintray.com/kotlin/exposed")
}

java {
    sourceCompatibility = JavaVersion.VERSION_1_8
}

kotlin {
    experimental.coroutines = Coroutines.ENABLE
}

dependencies {
    val kotlinVersion: String by extra
    val ktorVersion = "0.9.0"
    val exposedVersion = "0.9.1"
    val h2Version = "1.4.196"

    compile("org.jetbrains.kotlin:kotlin-stdlib-jre8:$kotlinVersion")
    testCompile("junit", "junit", "4.12")

    compile(kotlin("stdlib"))

    compile("io.ktor:ktor-server-netty:$ktorVersion")
    compile("io.ktor:ktor-gson:$ktorVersion")

    compile("org.jetbrains.exposed:exposed:$exposedVersion")
    compile("com.h2database:h2:$h2Version")

    compile("ch.qos.logback:logback-classic:1.2.1")
}
```

`gradle.properties` :
```kotlin
org.gradle.script.lang.kotlin.accessors.auto=true
kotlinVersion='1.2.10'
```

元のファイルはこちら

`build.gradle` :
```groovy
group 'jp.example.geckour'
version '1.0-SNAPSHOT'

buildscript {
    ext.kotlin_version = '1.2.10'
    ext.ktor_version = '0.9.0'
    ext.moshi_version = '1.5.0'

    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

apply plugin: 'java'
apply plugin: 'kotlin'
apply plugin: 'application'

mainClassName = 'io.ktor.server.netty.DevelopmentEngine'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
    maven { url "https://dl.bintray.com/kotlin/kotlinx" }
    maven { url  "http://dl.bintray.com/kotlin/ktor" }
    maven { url  "http://dl.bintray.com/kotlin/exposed" }
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jre8:$kotlin_version"
    testImplementation group: 'junit', name: 'junit', version: '4.12'

    implementation "io.ktor:ktor-server-netty:$ktor_version"
    implementation "io.ktor:ktor-gson:$ktor_version"

    implementation 'org.jetbrains.exposed:exposed:0.8.7'
    implementation 'com.h2database:h2:1.4.196'

    implementation 'ch.qos.logback:logback-classic:1.2.1'
}

compileKotlin {
    kotlinOptions.jvmTarget = "1.8"
}
compileTestKotlin {
    kotlinOptions.jvmTarget = "1.8"
}

kotlin {
    experimental {
        coroutines "enable"
    }
}
```

### 解説
要るかどうかわからないのがアレですが、まあ見て分かる通り割と分かりやすいです。  
まだ仕様が固まってないっぽいので情報の参照先によって書き方が違う、とかその通りに書いたら怒られる、とかあってなかなかハードモードでした。  
Lintエラーが出てる時に、文法が間違ってるのかLintチェッカが息してないのかわからないのもポイント。

現場からは以上です。
