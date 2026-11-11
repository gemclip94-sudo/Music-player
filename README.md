name: Build Minimalist Player APK

on:
  push:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Generate Android Project Files
        run: |
          mkdir -p app/src/main/java/com/example/minimalistplayer

          cat << 'EOF' > settings.gradle.kts
          pluginManagement {
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
          include(":app")
          EOF

          cat << 'EOF' > build.gradle.kts
          plugins {
              id("com.android.application") version "8.4.0" apply false
              id("org.jetbrains.kotlin.android") version "1.9.23" apply false
          }
          EOF

          cat << 'EOF' > gradle.properties
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=true
          kotlin.code.style=official
          android.nonTransitiveRClass=true
          EOF

          cat << 'EOF' > app/build.gradle.kts
          plugins {
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
          }
          EOF

          cat << 'EOF' > app/src/main/AndroidManifest.xml
          <?xml version="1.0" encoding="utf-8"?>
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
          </manifest>
          EOF

          cat << 'EOF' > app/src/main/java/com/example/minimalistplayer/Track.kt
          package com.example.minimalistplayer
          data class Track(val id: String, val title: String, val artist: String, val durationMs: Long, val mediaUri: String)
          val sampleTracks = listOf(
              Track("1", "Weightless", "Marconi Union", 372000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3"),
              Track("2", "Aura", "Bicep", 425000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-2.mp3"),
              Track("3", "Solaris", "Cliff Martinez", 344000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-3.mp3"),
              Track("4", "An Ending (Ascent)", "Brian Eno", 302000L, "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-4.mp3")
          )
          EOF

          cat << 'EOF' > app/src/main/java/com/example/minimalistplayer/MusicPlayerViewModel.kt
          package com.example.minimalistplayer
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
                      }
                  })
                  viewModelScope.launch {
                      while (true) {
                          if (player.isPlaying) {
                              _uiState.update { it.copy(currentPositionMs = player.currentPosition) }
                          }
                          delay(500)
                      }
                  }
                  if (_uiState.value.trackList.isNotEmpty()) {
                      selectTrack(_uiState.value.trackList.first(), autoPlay = false)
                  }
              }

              fun selectTrack(track: Track, autoPlay: Boolean = true) {
                  _uiState.update { it.copy(currentTrack = track, durationMs = track.durationMs) }
                  player.setMediaItem(MediaItem.fromUri(track.mediaUri))
                  player.prepare()
                  if (autoPlay) player.play()
              }

              fun togglePlayPause() = if (player.isPlaying) player.pause() else player.play()

              fun seekTo(positionMs: Long) {
                  player.seekTo(positionMs)
                  _uiState.update { it.copy(currentPositionMs = positionMs) }
              }

              fun playNext() {
                  val tracks = _uiState.value.trackList
                  val idx = tracks.indexOf(_uiState.value.currentTrack)
                  if (idx != -1 && idx < tracks.lastIndex) selectTrack(tracks[idx + 1])
                  else if (tracks.isNotEmpty()) selectTrack(tracks.first())
              }

              fun playPrevious() {
                  val tracks = _uiState.value.trackList
                  val idx = tracks.indexOf(_uiState.value.currentTrack)
                  if (idx > 0) selectTrack(tracks[idx - 1])
              }

              override fun onCleared() {
                  super.onCleared()
                  player.release()
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/example/minimalistplayer/MinimalistMusicPlayerScreen.kt
          package com.example.minimalistplayer
          import androidx.compose.animation.core.*
          import androidx.compose.foundation.background
          import androidx.compose.foundation.clickable
          import androidx.compose.foundation.layout.*
          import androidx.compose.foundation.lazy.LazyColumn
          import androidx.compose.foundation.lazy.itemsIndexed
          import androidx.compose.foundation.shape.CircleShape
          import androidx.compose.material.icons.Icons
          import androidx.compose.material.icons.rounded.*
          import androidx.compose.material3.*
          import androidx.compose.runtime.*
          import androidx.compose.ui.Alignment
          import androidx.compose.ui.Modifier
          import androidx.compose.ui.draw.clip
          import androidx.compose.ui.draw.rotate
          import androidx.compose.ui.graphics.Color
          import androidx.compose.ui.text.font.FontFamily
          import androidx.compose.ui.text.font.FontWeight
          import androidx.compose.ui.unit.dp
          import androidx.compose.ui.unit.sp
          import androidx.lifecycle.viewmodel.compose.viewModel

          @Composable
          fun MinimalistMusicPlayerScreen(viewModel: MusicPlayerViewModel = viewModel()) {
              val state by viewModel.uiState.collectAsState()
              Surface(modifier = Modifier.fillMaxSize(), color = Color(0xFF121212)) {
                  Column(
                      modifier = Modifier.fillMaxSize().padding(horizontal = 28.dp, vertical = 24.dp),
                      horizontalAlignment = Alignment.CenterHorizontally,
                      verticalArrangement = Arrangement.SpaceBetween
                  ) {
                      Text("NOW PLAYING", letterSpacing = 4.sp, fontSize = 11.sp, fontWeight = FontWeight.SemiBold, color = Color.White.copy(alpha = 0.4f), modifier = Modifier.padding(top = 16.dp))
                      MinimalVinylDisc(isPlaying = state.isPlaying)
                      Column(horizontalAlignment = Alignment.CenterHorizontally) {
                          Text(state.currentTrack?.title ?: "Select a Track", fontSize = 24.sp, fontWeight = FontWeight.Light, fontFamily = FontFamily.SansSerif, color = Color.White)
                          Spacer(modifier = Modifier.height(6.dp))
                          Text(state.currentTrack?.artist?.uppercase() ?: "", fontSize = 12.sp, letterSpacing = 2.sp, color = Color.White.copy(alpha = 0.5f))
                      }
                      Column(modifier = Modifier.fillMaxWidth(), horizontalAlignment = Alignment.CenterHorizontally) {
                          Slider(
                              value = if (state.durationMs > 0) state.currentPositionMs.toFloat() / state.durationMs.toFloat() else 0f,
                              onValueChange = { viewModel.seekTo((it * state.durationMs).toLong()) },
                              colors = SliderDefaults.colors(thumbColor = Color.White, activeTrackColor = Color.White, inactiveTrackColor = Color.White.copy(alpha = 0.15f))
                          )
                          Row(modifier = Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                              Text(formatTime(state.currentPositionMs), fontSize = 11.sp, color = Color.White.copy(alpha = 0.4f))
                              Text(formatTime(state.durationMs), fontSize = 11.sp, color = Color.White.copy(alpha = 0.4f))
                          }
                          Spacer(modifier = Modifier.height(16.dp))
                          Row(modifier = Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceEvenly, verticalAlignment = Alignment.CenterVertically) {
                              IconButton(onClick = { viewModel.playPrevious() }) { Icon(Icons.Rounded.SkipPrevious, "Previous", tint = Color.White, modifier = Modifier.size(32.dp)) }
                              Box(modifier = Modifier.size(68.dp).clip(CircleShape).background(Color.White).clickable { viewModel.togglePlayPause() }, contentAlignment = Alignment.Center) {
                                  Icon(if (state.isPlaying) Icons.Rounded.Pause else Icons.Rounded.PlayArrow, "Play/Pause", tint = Color.Black, modifier = Modifier.size(36.dp))
                              }
                              IconButton(onClick = { viewModel.playNext() }) { Icon(Icons.Rounded.SkipNext, "Next", tint = Color.White, modifier = Modifier.size(32.dp)) }
                          }
                      }
                      QueuePreview(state.trackList, state.currentTrack) { viewModel.selectTrack(it) }
                  }
              }
          }

          @Composable
          fun MinimalVinylDisc(isPlaying: Boolean) {
              val transition = rememberInfiniteTransition(label = "disc")
              val angle by transition.animateFloat(0f, 360f, infiniteRepeatable(tween(12000, easing = LinearEasing)), label = "angle")
              Box(modifier = Modifier.size(220.dp).rotate(if (isPlaying) angle else 0f).clip(CircleShape).background(Color(0xFF1E1E1E)), contentAlignment = Alignment.Center) {
                  Box(modifier = Modifier.size(170.dp).clip(CircleShape).background(Color(0xFF161616)))
                  Box(modifier = Modifier.size(54.dp).clip(CircleShape).background(Color(0xFF282828)), contentAlignment = Alignment.Center) {
                      Box(modifier = Modifier.size(14.dp).clip(CircleShape).background(Color(0xFF121212)))
                  }
              }
          }

          @Composable
          fun QueuePreview(tracks: List<Track>, currentTrack: Track?, onTrackSelected: (Track) -> Unit) {
              Column(modifier = Modifier.fillMaxWidth().heightIn(max = 140.dp)) {
                  Text("PLAYLIST", fontSize = 10.sp, letterSpacing = 2.sp, color = Color.White.copy(alpha = 0.3f), modifier = Modifier.padding(bottom = 8.dp))
                  LazyColumn {
                      itemsIndexed(tracks) { index, track ->
                          val isSelected = track.id == currentTrack?.id
                          Row(modifier = Modifier.fillMaxWidth().clickable { onTrackSelected(track) }.padding(vertical = 6.dp), horizontalArrangement = Arrangement.SpaceBetween) {
                              Text("${index + 1}.  ${track.title}", fontSize = 13.sp, color = if (isSelected) Color.White else Color.White.copy(alpha = 0.5f))
                              Text(formatTime(track.durationMs), fontSize = 11.sp, color = Color.White.copy(alpha = 0.3f))
                          }
                      }
                  }
              }
          }

          fun formatTime(ms: Long): String {
              val s = (ms / 1000).coerceAtLeast(0)
              return "%02d:%02d".format(s / 60, s % 60)
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/example/minimalistplayer/MainActivity.kt
          package com.example.minimalistplayer
          import android.os.Bundle
          import androidx.activity.ComponentActivity
          import androidx.activity.compose.setContent
          import androidx.activity.enableEdgeToEdge

          class MainActivity : ComponentActivity() {
              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  enableEdgeToEdge()
                  setContent { MinimalistMusicPlayerScreen() }
              }
          }
          EOF

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3
        with:
          gradle-version: '8.7'

      - name: Build Debug APK
        run: gradle assembleDebug

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: MinimalistMusicPlayer-APK
          path: app/build/outputs/apk/debug/app-debug.apk
