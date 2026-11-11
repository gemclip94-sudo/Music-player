name: Build APK

on:
  push:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Generate Project Files
        shell: python
        run: |
          import os
          os.makedirs("app/src/main/java/com/example/minimalistplayer", exist_ok=True)

          files = {
              "settings.gradle.kts": '''pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "MinimalistMusicPlayer"
          include(":app")''',

              "build.gradle.kts": '''plugins {
              id("com.android.application") version "8.4.0" apply false
              id("org.jetbrains.kotlin.android") version "1.9.23" apply false
          }''',

              "gradle.properties": '''org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=true
          kotlin.code.style=official
          android.nonTransitiveRClass=true''',

              "app/build.gradle.kts": '''plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.example.minimalistplayer"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.example.minimalistplayer"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
              buildFeatures { compose = true }
              composeOptions { kotlinCompilerExtensionVersion = "1.5.13" }
          }
          dependencies {
              implementation(platform("androidx.compose:compose-bom:2024.04.01"))
              implementation("androidx.compose.ui:ui")
              implementation("androidx.compose.ui:ui-graphics")
              implementation("androidx.compose.material3:material3")
              implementation("androidx.compose.material:material-icons-extended")
              implementation("androidx.media3:media3-exoplayer:1.3.1")
              implementation("androidx.media3:media3-session:1.3.1")
              implementation("androidx.media3:media3-ui:1.3.1")
              implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
              implementation("androidx.activity:activity-compose:1.9.0")
          }''',

              "app/src/main/AndroidManifest.xml": '''<?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.INTERNET" />
              <uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />
              <application
                  android:allowBackup="true"
                  android:label="Minimalist Player"
                  android:supportsRtl="true"
                  android:theme="@android:style/Theme.DeviceDefault.NoActionBar">
                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
              </application>
          </manifest>''',

              "app/src/main/java/com/example/minimalistplayer/Track.kt": '''package com.example.minimalistplayer
          data class Track(val id: String, val title: String, val artist: String, val durationMs: Long, val mediaUri: String)
          val sampleTracks = listOf(
              Track("1", "Weightless", "Marconi Union", 372000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3"),
              Track("2", "Aura", "Bicep", 425000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-2.mp3"),
              Track("3", "Solaris", "Cliff Martinez", 344000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-3.mp3"),
              Track("4", "An Ending (Ascent)", "Brian Eno", 302000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-4.mp3")
          )''',

              "app/src/main/java/com/example/minimalistplayer/MusicPlayerViewModel.kt": '''package com.example.minimalistplayer
          import android.app.Application
          import androidx.lifecycle.AndroidViewModel
          import androidx.lifecycle.viewModelScope
          import androidx.media3.common.MediaItem
          import androidx.media3.common.Player
          import androidx.media3.exoplayer.ExoPlayer
          import kotlinx.coroutines.delay
          import kotlinx.coroutines.flow.*
          import kotlinx.coroutines.launch

          data class PlayerUiState(
              val currentTrack: Track? = null,
              val isPlaying: Boolean = false,
              val currentPositionMs: Long = 0L,
              val durationMs: Long = 0L,
              val trackList: List<Track> = sampleTracks
          )

          class MusicPlayerViewModel(application: Application) : AndroidViewModel(application) {
              private val player: ExoPlayer = ExoPlayer.Builder(application).build()
              private val _uiState = MutableStateFlow(PlayerUiState())
              val uiState: StateFlow<PlayerUiState> = _uiState.asStateFlow()

              init {
                  player.addListener(object : Player.Listener {
                      override fun onIsPlayingChanged(isPlaying: Boolean) {
                          _uiState.update { it.copy(isPlaying = isPlaying) }
                      }
                      override fun onPlaybackStateChanged(playbackState: Int) {
                          if (playbackState == Player.STATE_READY) {
                              _uiState.update { it.copy(durationMs = player.duration.coerceAtLeast(0L)) }
                          }
