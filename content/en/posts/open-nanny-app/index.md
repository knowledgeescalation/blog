---
title: Open Nanny App
date: 2025-04-01
author: Knowledge Escalation
description: Building a Smart Baby Monitor App with Kotlin and Jetpack Compose
isStarred: false
---

# Building a Smart Baby Monitor App with Kotlin and Jetpack Compose: A Comprehensive Tutorial

{{< justify >}}
In an era where smart home technology is becoming increasingly prevalent, creating a connected baby monitor app represents both a practical and educational project for Android developers. This tutorial walks through how to build an advanced baby monitor application - [OpenNannyApp](https://github.com/knowledgeescalation/OpenNannyApp) - that provides real-time video streaming, environmental sensor monitoring, music playback, and lighting control. We'll explore how to implement these features using modern Android development techniques including Kotlin, Jetpack Compose, Retrofit, and WebRTC.
{{< /justify >}}
## Introduction to the OpenNannyApp Architecture

{{< justify >}}
OpenNannyApp follows a modern MVVM (Model-View-ViewModel) architecture pattern, with a clear separation of concerns between UI components, business logic, and data handling. Before diving into implementation details, let's understand the app's core components:
{{< /justify >}}


1. **Network Communication Layer**: Handles API requests using Retrofit and token-based authentication
2. **Video Streaming Module**: Implements WebRTC for real-time video transmission
3. **UI Layer**: Built with Jetpack Compose for a reactive and declarative interface
4. **ViewModels**: Manage data flow and state for different features (sensors, music, lighting)

{{< justify >}}
This architecture promotes maintainability, testability, and separation of concerns—principles that are essential for any production-quality application.
{{< /justify >}}

## Setting Up the Network Communication Layer

{{< justify >}}
At the heart of our app is the ability to communicate with the baby monitor device. Let's examine how the [NetworkModule](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/api/NetworkModule.kt) establishes this critical connection.
{{< /justify >}}

### Implementing the API Service Interface

The first step is defining our API endpoints through a Retrofit interface:

```kotlin
interface ApiService {
    @GET("/sensors")
    suspend fun getSensors(): InfoObject
    
    @FormUrlEncoded
    @POST("/token")
    suspend fun login(
        @Field("username") username: String,
        @Field("password") password: String
    ): LoginResponse
    
    @POST("/led")
    suspend fun led(@Body user: LedRequest): LedResponse
    
    @POST("/music")
    suspend fun getDirs(@Body user: MusicRequest): DirObject
    
    // Additional music control endpoints...
}
```

{{< justify >}}
This interface defines all the endpoints our app will communicate with, using Retrofit annotations to specify the HTTP method, path, and parameter format.
{{< /justify >}}
### Token Authentication Implementation

To ensure secure communication, we implement token-based authentication:

```kotlin
class TokenAuthenticator(
    private val apiService: ApiService,
    private val getCredentials: () -> LoginRequest,
    private val saveToken: (String) ->; Unit
) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        // Retry logic for authentication
        if (responseCount(response) >= 3) {
            return null // Give up after 3 attempts
        }
        
        // Refresh token implementation
        return try {
            val newToken = runBlocking {
                val credentials = getCredentials()
                val loginResponse = apiService.login(credentials.username, credentials.password)
                loginResponse.token
            }
            
            saveToken(newToken)
            response.request.newBuilder()
                .header("Authorization", "Bearer $newToken")
                .build()
        } catch (e: Exception) {
            null
        }
    }
}
```

{{< justify >}}
This authenticator automatically refreshes expired tokens, providing seamless authentication handling.
{{< /justify >}}
### Creating the Network Module

{{< justify >}}
The `NetworkModule` ties everything together by creating an OkHttpClient with the authenticator and initializing the Retrofit instance:
{{< /justify >}}
```kotlin
class NetworkModule(val api_ip: String, val api_user: String, val api_pass: String) {
    private var token = ""
    val serviceAPI = createApiService()
    
    fun createApiService(): ApiService {
        val okHttpClient = OkHttpClient.Builder()
            .authenticator(TokenAuthenticator(
                apiService = /* bootstrap API service */,
                getCredentials = { LoginRequest(api_user, api_pass) },
                saveToken = { token -> this.saveToken(token) }
            ))
            .addInterceptor { chain ->
                // Add authorization header to requests
                val request = chain.request().newBuilder()
                    .header("Authorization", "Bearer $token")
                    .build()
                chain.proceed(request)
            }
            .build()
            
        return Retrofit.Builder()
            .baseUrl("https://$api_ip")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ApiService::class.java)
    }
}
```

{{< justify >}}
This design handles authentication transparently through interceptors and authenticators, allowing the rest of the application to focus on business logic rather than authentication details.
{{< /justify >}}
## Implementing Real-Time Video Streaming with WebRTC

{{< justify >}}
One of the most complex features of a baby monitor app is real-time video streaming. Our implementation uses WebRTC, a powerful technology for peer-to-peer communication.
{{< /justify >}}
### Setting Up WebRTC Components

We begin by initializing the WebRTC components in our [WebRTC](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/api/WebRTC.kt) class:

```kotlin
class WebRTC(private val context: Context, private val networkModule: NetworkModule, private val api_ip: String) {
    private val _remoteVideoTrackFlow = MutableSharedFlow<VideoTrack>()
    val remoteVideoTrackFlow: SharedFlow<VideoTrack> = _remoteVideoTrackFlow
    
    private val _remoteAudioTrackFlow = MutableSharedFlow<AudioTrack>()
    val remoteAudioTrackFlow: SharedFlow<AudioTrack> = _remoteAudioTrackFlow
    
    val eglBaseContext: EglBase.Context by lazy {
        EglBase.create().eglBaseContext
    }
    
    private val factory by lazy {
        PeerConnectionFactory.initialize(/* initialization options */)
        PeerConnectionFactory.builder()
            .setVideoDecoderFactory(DefaultVideoDecoderFactory(eglBaseContext))
            .createPeerConnectionFactory()
    }
    
    private val peerConnection = factory.createPeerConnection(
        emptyList(),
        object : PeerConnection.Observer {
            // Implementation of PeerConnection callbacks
            override fun onTrack(transceiver: RtpTransceiver) {
                val track = transceiver.receiver.track()
                if (track is VideoTrack) {
                    sessionManagerScope.launch {
                        _remoteVideoTrackFlow.emit(track)
                    }
                } else if (track is AudioTrack) {
                    sessionManagerScope.launch {
                        _remoteAudioTrackFlow.emit(track)
                    }
                }
            }
            // Other callback implementations...
        }
    )
}
```

{{< justify >}}
This setup initializes the WebRTC peer connection and provides flows for video and audio tracks that can be observed in the UI.
{{< /justify >}}
### Implementing WebSocket Signaling

{{< justify >}}
WebRTC requires a signaling mechanism to establish connections. We implement this using WebSockets:
{{< /justify >}}

```kotlin
private val client = OkHttpClient()
private val request = Request.Builder()
    .url("wss://$api_ip/webrtc")
    .header("Authorization", "Bearer ${networkModule.getToken()}")
    .build()
private var ws = client.newWebSocket(request, SignalingWebSocketListener())

private inner class SignalingWebSocketListener : WebSocketListener() {
    override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
        // Handle reconnection if authentication fails
        if (response?.code == 403) {
            CoroutineScope(Dispatchers.IO).launch {
                delay(1000)
                reconnectWebSocket()
            }
        }
    }
    
    override fun onMessage(webSocket: WebSocket, text: String) {
        offer = text
    }
}
```

{{< justify >}}
Our implementation automatically reconnects when authentication fails, ensuring a reliable connection.
{{< /justify >}}
### Establishing the WebRTC Connection

When an offer is received from the server, we establish the WebRTC connection:

```kotlin
private fun sendAnswer() {
	val json = offer?.let { JSONObject(it) }
	if (json != null && json.has("sdp")) {
        val sdp = SessionDescription(
            SessionDescription.Type.OFFER,
            json.getString("sdp")
        )
        
        peerConnection.setRemoteDescription(object : SdpObserver {
            override fun onSetSuccess() {
                peerConnection.createAnswer(object : SdpObserver {
                    override fun onCreateSuccess(answer: SessionDescription?) {
                        peerConnection.setLocalDescription(this, answer)
                        ws.send(answer?.description.toString())
                    }
                    // Other callback implementations...
                }, MediaConstraints())
            }
            // Other callback implementations...
        }, sdp)
    }
}
```

{{< justify >}}
This code accepts an offer from the server, creates an appropriate answer, and sends it back through the WebSocket to establish the peer-to-peer connection.
{{< /justify >}}
## Building the UI with Jetpack Compose

{{< justify >}}
Jetpack Compose provides a modern, declarative way to build UI. Let's look at how we implement the [main screen](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/MainActivity.kt) and navigation.
{{< /justify >}}
### Creating the Main Navigation Screen

The main screen provides navigation to different features:

```kotlin
@Composable
fun HomeScreen() {
    val context = LocalContext.current
    
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "Open Nanny",
            fontSize = 30.sp,
            fontWeight = FontWeight.Bold,
        )
        
        Spacer(modifier = Modifier.height(80.dp))
        
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            NavigationButton(
                text = "Sensors",
                imageRes = R.drawable.sensor,
                onClick = { context.startActivity(Intent(context, SensorsActivity::class.java)) }
            )
            
            NavigationButton(
                text = "Video",
                imageRes = R.drawable.video,
                onClick = { context.startActivity(Intent(context, VideoActivity::class.java)) }
            )
            
            NavigationButton(
                text = "Music",
                imageRes = R.drawable.music,
                onClick = { context.startActivity(Intent(context, MusicActivity::class.java)) }
            )
        }
    }
}
```

This creates a clean, centered navigation UI with icons and text labels for each main feature.

### Implementing the Sensors Screen

The [sensors screen](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/SensorsActivity.kt) displays environmental data from the baby monitor:

```kotlin
@Composable
fun SensorsScreen(viewModel: SensorsViewModel) {
    val state = viewModel.state.collectAsState()
    
    Box(contentAlignment = Alignment.Center) {
        when(val stateValue = state.value) {
            is InfoState.Error -> {
                Text(text = stateValue.message, fontSize = 24.sp, color = Color.White)
            }
            
            is InfoState.Success -> {
                LazyColumn {
                    items(stateValue.list) {
                        PrintInfoItem(it)
                    }
                }
            }
            
            InfoState.Loading -> {
                CircularProgressIndicator()
            }
        }
        
        Column(verticalArrangement = Arrangement.Bottom, horizontalAlignment = Alignment.CenterHorizontally) {
            IconButton(onClick = {viewModel.loadData()}, modifier = Modifier.size(64.dp)) {
                Image(
                    painter = painterResource(id = R.drawable.restart),
                    contentDescription = "Restart"
                )
            }
        }
    }
}
```

{{< justify >}}
This screen handles different states (loading, success, error) and displays sensor data in a scrollable list, with a refresh button at the bottom.
{{< /justify >}}
## Implementing the ViewModel Layer

{{< justify >}}
The ViewModel layer is responsible for managing UI state and handling business logic. Let's look at the implementation for sensors and lighting control.
{{< /justify >}}
### Sensors ViewModel

```kotlin
class SensorsViewModel(
    private val api: ApiService
) : ViewModel() {
    val state = MutableStateFlow<InfoState>(InfoState.Loading)
    
    init {
        loadData()
    }
    
    fun loadData() {
        viewModelScope.launch(Dispatchers.IO) {
            state.value = InfoState.Loading
            try {
                val info = api.getSensors()
                state.value = InfoState.Success(info.map { 
                    SensorsViewItem(name = it.name, value = it.value) 
                })
            } catch (exception: Exception) {
                state.value = InfoState.Error(message = exception.message.orEmpty())
            }
        }
    }
}
```

{{< justify >}}
[This](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/ui/viewModels/SensorsViewModel.kt) ViewModel fetches sensor data from the API and updates the UI state accordingly, handling errors gracefully.
{{< /justify >}}
### LED Control ViewModel

```kotlin
class LedViewModel(
    private val api: ApiService
) : ViewModel() {
    val initState = MutableStateFlow<LedState>(LedState.Loading)
    val state = MutableStateFlow<LedState>(LedState.Success(""))
    
    init {
        getStatus()
    }
    
    fun sendCmd(cmd: String) {
        viewModelScope.launch(Dispatchers.IO) {
            state.value = LedState.Loading
            try {
                val status = api.led(LedRequest(cmd))
                state.value = LedState.Success(status.status)
            } catch (exception: Exception) {
                state.value = LedState.Error(message = exception.message.orEmpty())
            }
        }
    }
    
    private fun getStatus() {
        viewModelScope.launch(Dispatchers.IO) {
            initState.value = LedState.Loading
            try {
                val status = api.led(LedRequest("status"))
                initState.value = LedState.Success(status.status)
            } catch (exception: Exception) {
                initState.value = LedState.Error(message = exception.message.orEmpty())
            }
        }
    }
}
```

{{< justify >}}
[This](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/ui/viewModels/LedViewModel.kt) ViewModel manages the lighting state, allowing for toggling between day and night modes.
{{< /justify >}}
## Creating the Video Screen

The [video screen](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/VideoActivity.kt) combines WebRTC video streaming with lighting controls:

```kotlin
@Composable
fun VideoScreen(webRTCsession: WebRTC, viewModel: LedViewModel) {
    var sliderPosition by remember { mutableStateOf(0f) }
    var isOn by remember { mutableStateOf(false) }
    val initState = viewModel.initState.collectAsState()
    val state = viewModel.state.collectAsState()
    
    LaunchedEffect(Unit) {
        webRTCsession.onSessionScreenReady()
    }
    
    DisposableEffect(Unit) {
        onDispose {
            webRTCsession.destroy()
        }
    }
    
    Box(modifier = Modifier.fillMaxSize()) {
        val remoteVideoTrackState by webRTCsession.remoteVideoTrackFlow.collectAsStateWithLifecycle(null)
        val remoteVideoTrack = remoteVideoTrackState
        val remoteAudioTrackState by webRTCsession.remoteAudioTrackFlow.collectAsStateWithLifecycle(null)
        val remoteAudioTrack = remoteAudioTrackState
        
        if (remoteAudioTrack != null) {
            AudioRenderer(audioTrack = remoteAudioTrack)
        }
        
        if (remoteVideoTrack != null) {
            VideoRenderer(
                eglContext = webRTCsession.eglBaseContext,
                videoTrack = remoteVideoTrack,
                modifier = Modifier.align(Alignment.Center)
            )
        } else {
            Column(verticalArrangement = Arrangement.Bottom, modifier = Modifier.align(Alignment.Center)) {
                CircularProgressIndicator()
            }
        }
        
        // LED control UI implementation
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(bottom = 32.dp),
            contentAlignment = Alignment.BottomCenter
        ) {
            // Implementation of day/night mode slider control
            when(val stateInitValue = initState.value) {
                is LedState.Success -&gt; {
                    // Update slider based on current LED state
                    if(stateInitValue.status == "day") {
                        sliderPosition = 0f
                        isOn = false
                    }
                    if(stateInitValue.status == "night") {
                        sliderPosition = 1f
                        isOn = true
                    }
                    
                    // Slider UI for controlling day/night mode
                    Row(verticalAlignment = Alignment.CenterVertically) {
                        // UI implementation for day/night slider
                        Image(painter = painterResource(id = R.drawable.sun), contentDescription = "sun")
                        Slider(
                            value = sliderPosition,
                            onValueChange = { /* update slider position */ },
                            onValueChangeFinished = {
                                isOn = sliderPosition &gt; 0.5f
                                sliderPosition = if (isOn) 1f else 0f
                                if(isOn) {
                                    viewModel.sendCmd("night")
                                } else {
                                    viewModel.sendCmd("day")
                                }
                            }
                        )
                        Image(painter = painterResource(id = R.drawable.moon), contentDescription = "moon")
                    }
                }
                // Handle loading and error states
            }
        }
    }
}
```

{{< justify >}}
This screen integrates both WebRTC video rendering and lighting controls in a unified interface, demonstrating how to combine multiple features in a single screen.
{{< /justify >}}
## Building the Music Player

The music player allows playing lullabies and other audio content for the baby.

### Implementing the Music Selection Screen

First, we create a [directory browser](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/MusicActivity.kt#L137) to select music:

```kotlin
@Composable
fun DirectoriesScreen(
    dirState: DirState,
    onDirectorySelected: (String) -> Unit
) {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(
                text = "Music",
                style = MaterialTheme.typography.headlineMedium
            )
            
            Spacer(modifier = Modifier.height(40.dp))
            
            when (dirState) {
                is DirState.Success -> {
                    DirectoryList(
                        directories = dirState.list,
                        onDirectorySelected = onDirectorySelected
                    )
                }
                // Handle loading and error states
            }
        }
    }
}
```

This screen shows available music directories and handles navigation to individual songs.

### Implementing the Music Player Controls

The [music player](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/MusicActivity.kt#L387) screen provides playback controls:

```kotlin
@Composable
fun MediaPlayerScreen(
    directory: String,
    songName: String,
    onStop: () -> Unit,
    viewModel: MusicViewModel
) {
    val songState by viewModel.songState.collectAsState()
    var volume by remember { mutableFloatStateOf(0.0f) }
    var progress by remember { mutableFloatStateOf(0.0f) }
    var progressTxt by remember { mutableLongStateOf(0.toLong()) }
    
    // Start playback when screen is shown if not already playing
    LaunchedEffect(songName) {
        if (songState !is SongState.Playing) {
            viewModel.playSong(songName=songName, directory = directory)
        }
        
        // Update progress periodically
        while (true) {
            delay(1000)
            if (songState is SongState.Playing &amp;&amp; !(songState as SongState.Playing).isPaused) {
                val myDuration = (songState as SongState.Playing).duration
                if (progressTxt < myDuration) {
                    progressTxt += 1
                    progress = progressTxt.toFloat() / myDuration.toFloat()
                }
            }
        }
    }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // Player UI implementation with progress bar, play/pause/stop buttons, and volume control
        
        when (songState) {
            is SongState.Playing -> {
                // Display progress bar
                Slider(
                    value = progress,
                    onValueChange = { /* update progress UI */ },
                    onValueChangeFinished = { viewModel.rewindSong(progressTxt) }
                )
                
                // Display playback controls
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceEvenly
                ) {
                    // Stop button
                    IconButton(onClick = {
                        viewModel.stopSong()
                        onStop()
                    }) {
                        Icon(imageVector = Icons.Default.Stop, contentDescription = "Stop")
                    }
                    
                    // Play/Pause button
                    IconButton(onClick = {
                        if ((songState as SongState.Playing).isPaused) {
                            viewModel.resumeSong()
                        } else {
                            viewModel.pauseSong()
                        }
                    }) {
                        Icon(
                            imageVector = if ((songState as SongState.Playing).isPaused)
                                Icons.Default.PlayArrow else Icons.Default.Pause,
                            contentDescription = if ((songState as SongState.Playing).isPaused) 
                                "Resume" else "Pause"
                        )
                    }
                }
                
                // Volume control
                Slider(
                    value = volume,
                    onValueChange = { volume = it },
                    onValueChangeFinished = { viewModel.setVolume(volume) }
                )
            }
            // Handle other states
        }
    }
}
```

{{< justify >}}
This implementation provides a complete music player with progress tracking, play/pause/stop controls, and volume adjustment.
{{< /justify >}}
## Implementing the Music ViewModel

The [Music](https://github.com/knowledgeescalation/OpenNannyApp/blob/master/src/main/java/com/example/opennannyapp/ui/viewModels/MusicViewModel.kt#L16) ViewModel manages the complex state of music playback:

```kotlin
class MusicViewModel(private val api: ApiService) : ViewModel() {
    val dirState = MutableStateFlow<DirState>(DirState.Loading)
    val mp3State = MutableStateFlow<Mp3State>(Mp3State.Loading)
    val songState = MutableStateFlow<SongState>(SongState.Initial)
    
    fun playSong(songName: String, directory: String) {
        viewModelScope.launch(Dispatchers.IO) {
            songState.value = SongState.Loading
            try {
                val response = api.songPlay(MusicRequest(cmd = "play", parameters = "$directory/$songName.mp3"))
                if (response.status == "OK") {
                    songState.value = SongState.Playing(
                        songName = songName,
                        directory = directory,
                        duration = response.duration ?: 0L,
                        currentProgress = 0L,
                        volume = response.volume ?: 0.0f
                    )
                    startStatusUpdates()
                } else {
                    songState.value = SongState.Error(message = response.status)
                }
            } catch (e: Exception) {
                songState.value = SongState.Error(message = e.message.orEmpty())
            }
        }
    }
    
    // Implement pause, resume, stop, rewind, and volume control methods
    
    private fun startStatusUpdates() {
        viewModelScope.launch(Dispatchers.IO) {
            while (songState.value is SongState.Playing && !(songState.value as SongState.Playing).isPaused) {
                delay(5000) // Update every 5 seconds
                updateSongStatus()
            }
        }
    }
    
    suspend fun updateSongStatus() {
        try {
            val response = api.songStatus(MusicRequest(cmd = "status", parameters = ""))
            // Update state based on response
        } catch (exception: Exception) {
            // Handle errors
        }
    }
}
```

{{< justify >}}
This ViewModel manages the complex state transitions of music playback and provides periodic status updates to keep the UI in sync with the actual playback state on the server.
{{< /justify >}}
## Application Initialization

The app initialization ties everything together:

```kotlin
class OpenNannyApp : Application() {
    lateinit var networkModule: NetworkModule
    lateinit var serviceApi: ApiService
    lateinit var api_ip: String
    
    override fun onCreate() {
        super.onCreate()
        api_ip = getString(R.string.api_ip)
        val api_user = getString(R.string.api_user)
        val api_pass = getString(R.string.api_pass)
        
        networkModule = NetworkModule(api_ip=api_ip, api_user=api_user, api_pass=api_pass)
        serviceApi = networkModule.serviceAPI
    }
}
```

This initializes the network components with credentials stored in string resources, making them available throughout the app.

## Conclusion: Putting It All Together

{{< justify >}}
Building a connected baby monitor app involves multiple complex components working together seamlessly. Through this tutorial, we've demonstrated how to:
{{< /justify >}}

1. Implement a secure API communication layer with token authentication
2. Set up real-time video streaming using WebRTC
3. Create a responsive UI with Jetpack Compose
4. Build feature-specific ViewModels for state management
5. Implement a music player with full playback control
6. Add environmental monitoring through sensor data display
7. Include lighting control with day/night modes

{{< justify >}}
The architecture we've built is both robust and extensible. You could further enhance this application by adding:
{{< /justify >}}

- Push notifications for important events (temperature alerts, noise detection)
- Historical data logging and visualization
- Multiple camera support
- Custom lullaby playlists
- Integration with smart home platforms

{{< justify >}}
By following modern Android development practices and leveraging powerful libraries like Retrofit, WebRTC, and Jetpack Compose, we've created a sophisticated yet maintainable application that demonstrates how to build complex connected systems.

The complete implementation showcases the power of Kotlin coroutines for asynchronous operations, Flow for reactive programming, and MVVM architecture for clean separation of concerns - all essential skills for modern Android development.
{{< /justify >}}


