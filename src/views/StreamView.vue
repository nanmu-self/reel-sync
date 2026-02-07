<template>
  <div class="stream-root">
    <div class="stream-container">
      <div class="header-section">
        <div class="title-group">
          <h1 style="font-weight: bold;">{{ $t("StreamView.title") }}</h1>
          <div id="status">
            <span class="status-dot" :style="{ color: isReady ? '#4caf50' : '#f44336' }">⬤</span>
            {{ isReady ? $t("StreamView.messages.connected") : $t("StreamView.messages.disconnected") }}
          </div>
        </div>

        <!-- Voice Toggle -->
        <div v-if="isReady" class="voice-controls">
          <mdui-switch id="voice-switch" ref="voiceSwitch" @change="toggleVoice" :checked="isVoiceEnabled"
            checked-icon="videocam--rounded" unchecked-icon="videocam_off--rounded"></mdui-switch>
          <label>{{ $t("StreamView.labels.voiceToggle") }}</label>
        </div>
      </div>

      <mdui-card variant="outlined" class="video-card">
        <reelsync-video-player id="video-player-stream" ref="videoPlayerStream"
          v-show="isReady"></reelsync-video-player>

        <div v-if="!isReady" class="loading-overlay">
          <reelsync-loading-ring id="loading"></reelsync-loading-ring>
          <h3>{{ loadingDescription }}</h3>
          <p v-if="isConnectionRestricted" class="error-text">
            {{ $t("StreamView.messages.connectionRestricted") }}
          </p>
          <p class="hint-text">{{ hint }}</p>
        </div>
      </mdui-card>

      <div class="info-section">
        <mdui-card variant="outlined" clickable class="info-group" @click="copyRoomID">
          <span class="label">{{ $t("StreamView.messages.roomID", { roleDescription, roomID: "" }) }}</span>
          <span class="value monospace">{{ roomID }}</span>
        </mdui-card>
        <vue-qrcode :color="{ dark: '#000000', light: '#ffffff' }" type="image/webp" :value="locationOrigin+'/?join='+roomID" />
        <mdui-card v-if="!isClient" variant="outlined" clickable class="info-group" @click="copyShareLink(false)">
          <span class="label">{{ $t("StreamView.messages.shareLink") }}</span>
          <span class="value monospace">{{ locationOrigin }}/?join={{ roomID }}</span>
        </mdui-card>

        <mdui-card v-if="isReady" variant="outlined" class="info-group">
          <span class="label">{{ $t("StreamView.messages.networkInfo") }}</span>
          <span class="value">
            {{
              !isClient
                ? $t("StreamView.messages.pushing", {
                  m: method == 1 ? $t("StreamView.messages.sameOriginLiteral") : $t("StreamView.messages.p2pLiteral"),
                })
                : $t("StreamView.messages.watching")
            }}<span v-if="playbackDelta !== null || rtt !== null" class="latency">
              ({{ method == 1 ? $t("StreamView.messages.delta") : $t("StreamView.messages.latency") }}:
              {{
                method == 1
                  ? playbackDelta !== null
                    ? Math.round(playbackDelta * 1e3) + "ms"
                    : $t("StreamView.messages.measuringLiteral")
                  : rtt !== null
                    ? Math.round(rtt * 1e3) + "ms"
                    : $t("StreamView.messages.measuringLiteral")
              }})
            </span>
          </span>
        </mdui-card>
      </div>

      <!-- Video Quality Selector (host P2P mode only) -->
      <mdui-card v-if="!isClient && method == 0" variant="elevated" class="quality-card">
        <span class="label">{{ $t("StreamView.labels.videoQuality") }}</span>
        <mdui-segmented-button-group selects="single" :value="shared.app.videoQuality" @change="onQualityChange"
          full-width class="quality-buttons">
          <mdui-segmented-button value="low">
            {{ $t("StreamView.qualityPresets.low") }}
          </mdui-segmented-button>
          <mdui-segmented-button value="medium">
            {{ $t("StreamView.qualityPresets.medium") }}
          </mdui-segmented-button>
          <mdui-segmented-button value="high">
            {{ $t("StreamView.qualityPresets.high") }}
          </mdui-segmented-button>
          <mdui-segmented-button value="ultra">
            {{ $t("StreamView.qualityPresets.ultra") }}
          </mdui-segmented-button>
          <mdui-segmented-button value="unlimited">
            {{ $t("StreamView.qualityPresets.unlimited") }}
          </mdui-segmented-button>
        </mdui-segmented-button-group>
        <span class="quality-description">{{ qualityDescription }}</span>
      </mdui-card>
    </div>
    <!-- Video Call Window -->
    <div 
      v-show="shared.app.isVoiceEnabled || remoteVoiceEnabled" 
      class="video-call-window"
      :class="{ collapsed: isCollapsed }"
      :style="videoWindowStyle"
      @mousedown="startDrag"
      @touchstart="startDrag"
      @click="handleWindowClick"
    >
      <template v-if="!isCollapsed">
        <video ref="remoteCallVideo" class="remote-call-video" autoplay playsinline></video>
        <video ref="localCallVideo" class="local-call-video" autoplay playsinline muted></video>
      </template>
      <div v-else class="collapsed-content">
        <mdui-icon name="videocam--rounded"></mdui-icon>
      </div>
    </div>
  </div>
</template>

<script>
import VueQrcode from 'vue-qrcode';
import { useSharedStore } from "@/stores/shared";
import { msg } from "@/utils/msg";
import { Comm } from "@/utils/comm";
import { createSdpTransform, applyBitrateLimit, applyTrackConstraints, QualityPresets } from "@/utils/video-quality";
import { alert as mduiAlert } from "mdui/functions/alert";
import { snackbar as mduiSnackbar } from "mdui/functions/snackbar";

import LoadingRing from "@/components/LoadingRing.vue";
import VideoPlayer from "@/components/VideoPlayer.vue";

export default {
  components: {
    "reelsync-loading-ring": LoadingRing,
    "reelsync-video-player": VideoPlayer,
    "vue-qrcode": VueQrcode
  },
  name: "StreamView",
  data() {
    return {
      connectionAttempts: 0,
      maxAttempts: 3,
      isReady: false,
      playbackDelta: null, // 同源模式下为同步偏差，点对点下为null
      rtt: null, // 点对点模式下为RTT，单位秒
      locationOrigin: location.origin,
      audioCallListenerRegistered: false, // 防止重复注册音频来电监听器
      remoteVoiceEnabled: false, // 追踪对方是否启用了语音
      reconnectTimer: null,
      shared: useSharedStore(),
      // Video window state
      windowPos: { x: 0, y: 70 },
      isDragging: false,
      dragOffset: { x: 0, y: 0 },
      isCollapsed: false,
      dragStartTime: 0,
      hasMoved: false,
    };
  },
  computed: {
    videoWindowStyle() {
      if (this.isCollapsed) {
        return {
          transform: `translate(${this.windowPos.x}px, ${this.windowPos.y}px)`,
          width: '48px',
          height: '48px',
          borderRadius: '24px',
          cursor: 'pointer',
          transition: 'all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1)'
        };
      }
      return {
        transform: `translate(${this.windowPos.x}px, ${this.windowPos.y}px)`,
        cursor: this.isDragging ? 'grabbing' : 'grab',
        transition: this.isDragging ? 'none' : 'all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1)'
      };
    },
    method() {
      return this.shared.app.method;
    },
    roleDescription() {
      return this.isClient
        ? this.shared.app.i18n.t("StreamView.roleDescription.client")
        : this.shared.app.i18n.t("StreamView.roleDescription.host");
    },
    hint() {
      return this.isClient
        ? this.shared.app.i18n.t("StreamView.hint.client")
        : this.shared.app.i18n.t("StreamView.hint.host");
    },
    loadingDescription() {
      return this.shared.app.mode == 1
        ? this.shared.app.i18n.t("StreamView.messages.joining")
        : this.shared.app.i18n.t("StreamView.messages.waiting");
    },
    roomID() {
      return this.isClient && this.shared.app.guestID ? this.shared.app.guestID : this.shared.app.roomID;
    },
    isClient() {
      return this.shared.app.mode == 1;
    },
    isConnectionRestricted() {
      return this.shared.app.isConnectionRestricted;
    },
    isVoiceEnabled() {
      return this.shared.app.isVoiceEnabled;
    },
    qualityDescription() {
      const preset = this.shared.app.videoQuality;
      const q = QualityPresets[preset];
      if (!q || preset === "unlimited") {
        return this.$t("StreamView.qualityDescriptions.unlimited");
      }
      const w = q.video.width?.ideal || "?";
      const h = q.video.height?.ideal || "?";
      const fps = q.video.frameRate?.ideal || "?";
      const mbps = (q.maxBitrate / 1_000_000).toFixed(1);
      return this.$t("StreamView.qualityDescriptions.detail", {
        resolution: `${w}×${h}`,
        fps,
        bitrate: `${mbps} Mbps`,
      });
    },
  },
  methods: {
    getVideoElement() {
      return this.$refs.videoPlayerStream?.$el ?? null;
    },
    onQualityChange(event) {
      const value = event?.target?.value;
      if (value && QualityPresets[value]) {
        this.shared.app.videoQuality = value;
        msg.i(`Video quality changed to: ${value}`);
      }
    },
    async toggleVoice(event) {
      const checked = event?.target?.checked ?? this.$refs.voiceSwitch?.checked;
      this.shared.app.isVoiceEnabled = !!checked;
      const comm = new Comm();

      if (this.shared.app.isVoiceEnabled) {
        try {
          // Check for getUserMedia support with fallback for older browsers
          const getUserMedia = navigator.mediaDevices?.getUserMedia?.bind(navigator.mediaDevices)
            || ((constraints) => new Promise((resolve, reject) => {
              const legacyGetUserMedia = navigator.getUserMedia
                || navigator.webkitGetUserMedia
                || navigator.mozGetUserMedia
                || navigator.msGetUserMedia;
              if (!legacyGetUserMedia) {
                reject(new Error("getUserMedia is not supported in this browser"));
                return;
              }
              legacyGetUserMedia.call(navigator, constraints, resolve, reject);
            }));

          // Request camera and microphone access
          this.shared.app.audioStream = await getUserMedia({
            audio: {
              echoCancellation: true,
              noiseSuppression: true,
              autoGainControl: true,
            },
            video: {
              width: { ideal: 320 },
              height: { ideal: 240 },
              frameRate: { ideal: 24 },
              facingMode: "user"
            },
          });

          msg.i("Camera and microphone access granted, starting call");
          
          // Display local video
          if (this.$refs.localCallVideo) {
            this.$refs.localCallVideo.srcObject = this.shared.app.audioStream;
          }

          // Setup call listener before notifying peer
          this.setupAudioCallListener();

          // Notify the remote peer about call enablement
          if (this.shared.peers.remote.data) {
            if (this.isClient) {
              this.shared.peers.remote.data.send(comm.client.voiceEnabled());
            } else {
              this.shared.peers.remote.data.send(comm.host.voiceEnabled());
            }
          }

          // Start call if remote peer has also enabled it
          if (this.remoteVoiceEnabled) {
            this.startAudioCall();
          }
        } catch (error) {
          msg.e("Failed to access media devices:", error);
          this.shared.app.isVoiceEnabled = false;
          if (this.$refs.voiceSwitch) this.$refs.voiceSwitch.checked = false;
          
          mduiSnackbar({
            message: this.$t("StreamView.messages.cameraError") + (error.message ? ` (${error.message})` : ""),
            placement: "top",
          });
        }
      } else {
        // Disable call
        this.stopAudioCall();

        // Notify the remote peer about call disablement
        if (this.shared.peers.remote.data) {
          if (this.isClient) {
            this.shared.peers.remote.data.send(comm.client.voiceDisabled());
          } else {
            this.shared.peers.remote.data.send(comm.host.voiceDisabled());
          }
        }

        msg.i("Video call disabled");
      }
    },

    setupAudioCallListener() {
      // Prevent duplicate registration
      if (this.audioCallListenerRegistered || !this.shared.peers.local.audio) return;
      this.audioCallListenerRegistered = true;

      this.shared.peers.local.audio.on("call", (call) => {
        msg.i("Incoming call");

        // Answer with local stream if available, otherwise answer without stream
        if (this.shared.app.audioStream) {
          call.answer(this.shared.app.audioStream);
        } else {
          // If no local stream, just answer to receive remote stream
          call.answer();
        }

        call.on("stream", (remoteStream) => {
          msg.i("Received remote stream from incoming call");
          this.playRemoteAudio(remoteStream);
        });

        call.on("error", (err) => {
          msg.e("Call error:", err);
        });

        call.on("close", () => {
          msg.i("Call closed");
        });

        this.shared.peers.remote.audio = call;
      });
    },

    playRemoteAudio(remoteStream) {
      // Create or reuse video element to play remote stream
      this.$nextTick(() => {
        const videoElement = this.$refs.remoteCallVideo;
        if (!videoElement) {
           console.warn("Video element not found in playRemoteAudio");
           return;
        }

        // For some browsers, we need to handle autoplay restrictions
        videoElement.srcObject = remoteStream;
        const playPromise = videoElement.play();
        if (playPromise !== undefined) {
          playPromise.catch((error) => {
            msg.w("Video autoplay was prevented:", error);
            // Video will play when user interacts with the page
          });
        }
      });
    },

    startAudioCall() {
      if (!this.shared.app.audioStream || !this.shared.peers.remote.data) {
        msg.w("Cannot start call: missing stream or data connection");
        return;
      }

      if (!this.shared.peers.local.audio) {
        msg.e("Call peer not initialized");
        return;
      }

      const peerID = this.isClient ? this.shared.app.roomID : this.shared.app.guestID;

      if (!peerID) {
        msg.e("Cannot start call: peer ID not available");
        return;
      }

      // Close existing call if any
      if (this.shared.peers.remote.audio) {
        this.shared.peers.remote.audio.close();
      }

      try {
        // Make a call to the remote peer
        const call = this.shared.peers.local.audio.call(`${peerID}-audio`, this.shared.app.audioStream);

        if (!call) {
          msg.e("Failed to initiate call");
          return;
        }

        call.on("stream", (remoteStream) => {
          msg.i("Received remote stream from outgoing call");
          this.playRemoteAudio(remoteStream);
        });

        call.on("error", (err) => {
          msg.e("Call error:", err);
        });

        call.on("close", () => {
          msg.i("Outgoing call closed");
        });

        this.shared.peers.remote.audio = call;
        msg.i(`Call initiated to ${peerID}-audio`);
      } catch (error) {
        msg.e("Failed to start call:", error);
      }
    },

    stopAudioCall() {
      // Stop local stream
      if (this.shared.app.audioStream) {
        this.shared.app.audioStream.getTracks().forEach((track) => track.stop());
        this.shared.app.audioStream = null;
      }

      // Close remote call
      if (this.shared.peers.remote.audio) {
        this.shared.peers.remote.audio.close();
        this.shared.peers.remote.audio = null;
      }

      // Remove remote video element source
      const remoteVideo = this.$refs.remoteCallVideo;
      if (remoteVideo) remoteVideo.srcObject = null;

      // Remove local video element source
      const localVideo = this.$refs.localCallVideo;
      if (localVideo) localVideo.srcObject = null;
    },
    copyShareLink(silent = false) {
      const link = `${this.locationOrigin}/?join=${this.roomID}`;
      navigator.clipboard.writeText(link).then(() => {
        if (!silent) {
          mduiSnackbar({
            message: this.shared.app.i18n.t("StreamView.messages.copiedToClipboard"),
          });
        }
      });
    },
    copyRoomID() {
      navigator.clipboard.writeText(this.roomID).then(() => {
        mduiSnackbar({
          message: this.shared.app.i18n.t("StreamView.messages.copiedToClipboard"),
        });
      });
    },
    connectToPeer() {
      const remotePeerId = `${this.shared.app.roomID}-data`;
      const dataPeer = this.shared.peers.local.data;

      if (!dataPeer) {
        msg.w("Connection aborted: dataPeer is null (possibly returning to home)");
        return;
      }

      const comm = new Comm();

      // 确保peer已经就绪
      if (dataPeer.disconnected) {
        dataPeer.reconnect();
      }

      msg.i(`Connecting to ${remotePeerId} (Attempt ${this.connectionAttempts + 1})`);

      // 建立数据通道连接
      const conn = dataPeer.connect(remotePeerId, {
        reliable: true,
      });

      // 连接成功回调
      conn.on("open", () => {
        conn.send(comm.client.greet(this.shared.app.guestID));
        msg.i(`Connection established with ${this.shared.app.roomID}.`);
        this.shared.peers.local.video.on("call", (call) => {
          call.answer();
          call.on("stream", (stream) => {
            msg.i("Received video stream");
            const video = this.getVideoElement();
            if (video) {
              video.srcObject = stream;
              video.load();
              video.play();
            }
          });
        });

        // Setup audio call listener (will be used when voice is enabled)
        this.setupAudioCallListener();
        this.isReady = true;
        this.shared.peers.remote.data = conn;
        // 启动RTT定时测量（点对点模式）
        if (this.shared.app.method == 0) {
          if (this.rttTimer) clearInterval(this.rttTimer);
          const rttInterval =
            (import.meta.env.VITE_LATENCY_MEASUREMENT_INTERVAL_SECONDS
              ? parseFloat(import.meta.env.VITE_LATENCY_MEASUREMENT_INTERVAL_SECONDS)
              : 2) * 1e3;
          this.rttTimer = setInterval(() => {
            conn.send(comm.client.rttPing(Date.now()));
            this._rttPingSent = Date.now();
          }, rttInterval);
        }
      });

      // 数据接收处理
      conn.on("data", (data) => {
        const commMsg = comm.resolve(data.toString());
        const video = this.getVideoElement();
        switch (commMsg.command) {
          case "origin":
            this.shared.app.method = 1;
            if (video) video.src = commMsg.data.ori;
            this.shared.app.syncThread = setInterval(
              () => {
                if (!video) return;
                msg.i(`Sending playback progress: ${video.currentTime}`);
                conn.send(comm.client.progress(video.currentTime, Date.now()));
              },
              import.meta.env.VITE_SAME_ORIGIN_SYNC_INTERVAL_SECONDS * 1e3,
            );
            break;
          case "play":
            video?.play();
            break;
          case "pause":
            video?.pause();
            break;
          case "seek":
            if (video) video.currentTime = commMsg.data.time;
            break;
          case "latency":
            this.playbackDelta = parseFloat(commMsg.data.lat);
            break;
          case "timestamp": {
            const timestamp = parseFloat(commMsg.data.atu);
            const latency = (Date.now() - timestamp) / 1000;
            this.playbackDelta = latency;
            conn.send(comm.client.latency(latency));
            break;
          }
          case "rtt-ping":
            // 对方发起RTT measurement，立即回复pong
            conn.send(comm.client.rttPong(commMsg.data.ts));
            break;
          case "rtt-pong":
            // 收到pong，计算RTT
            if (commMsg.data.ts && commMsg.data.ts2) {
              const sent = parseInt(commMsg.data.ts);
              const now = Date.now();
              // RTT = (now - sent) - (ts2 - sent)
              this.rtt = (now - sent) / 1000;
            }
            break;
          case "voice-enabled":
            msg.i("Remote peer enabled voice call");
            this.remoteVoiceEnabled = true;
            // Setup listener to receive incoming audio calls
            this.setupAudioCallListener();
            // If we also have voice enabled, initiate the call
            if (this.shared.app.isVoiceEnabled && this.shared.app.audioStream) {
              this.startAudioCall();
            }
            break;
          case "voice-disabled":
            msg.i("Remote peer disabled voice call");
            this.remoteVoiceEnabled = false;
            break;
          case "shutdown":
            msg.w("Remote peer shut down the session");
            mduiAlert({
              headline: this.$t("StreamView.messages.shutdown.headline"),
              description: this.$t("StreamView.messages.shutdown.description"),
              confirmText: this.$t("StreamView.messages.shutdown.confirmText"),
              onConfirm: () => {
                this.$router.replace("/");
              },
            });
            break;
          default:
            msg.w(`Invalid message: ${data}`);
            break;
        }
      });

      // 错误处理和重试
      conn.on("error", (err) => {
        msg.e(`Failed to establish connection with ${this.shared.app.roomID}: ${err.message}`);
        this.connectionAttempts++;
        // 延迟重试
        if (this.connectionAttempts < this.maxAttempts) {
          this.reconnectTimer = setTimeout(() => this.connectToPeer(), 1000);
        } else {
          msg.e("Max connection attempts reached. Please check the room ID and your connection.");
        }
      });

      // 连接关闭处理
      conn.on("close", () => {
        msg.w("Connection closed");
        this.shared.peers.remote.data = null;
        this.isReady = false;
        this.remoteVoiceEnabled = false;
        const video = this.getVideoElement();
        video?.pause();
        if (this.rttTimer) clearInterval(this.rttTimer);
        this.rtt = null;
        // Clean up voice call on disconnect
        this.stopAudioCall();
      });
    },
    handleResize() {
      // Ensure window stays within bounds on resize
      const maxX = window.innerWidth - (this.isCollapsed ? 48 : 240);
      const maxY = window.innerHeight - (this.isCollapsed ? 48 : 180);
      
      this.windowPos.x = Math.min(Math.max(0, this.windowPos.x), maxX);
      this.windowPos.y = Math.min(Math.max(0, this.windowPos.y), maxY);
    },
    startDrag(e) {
      // Prevent drag if targeting controls (if any)
      if (e.target.tagName === 'BUTTON' || e.target.tagName === 'MDUI-ICON') {
        // Allow click to pass through
        return;
      }
      
      this.isDragging = true;
      this.hasMoved = false;
      this.dragStartTime = Date.now();
      
      const clientX = e.clientX || e.touches[0].clientX;
      const clientY = e.clientY || e.touches[0].clientY;
      
      this.dragOffset.x = clientX - this.windowPos.x;
      this.dragOffset.y = clientY - this.windowPos.y;
      
      document.addEventListener('mousemove', this.onDrag);
      document.addEventListener('touchmove', this.onDrag, { passive: false });
      document.addEventListener('mouseup', this.stopDrag);
      document.addEventListener('touchend', this.stopDrag);
    },
    onDrag(e) {
      if (!this.isDragging) return;
      e.preventDefault(); // Prevent scrolling
      
      this.hasMoved = true;
      const clientX = e.clientX || (e.touches ? e.touches[0].clientX : 0);
      const clientY = e.clientY || (e.touches ? e.touches[0].clientY : 0);
      
      this.windowPos.x = clientX - this.dragOffset.x;
      this.windowPos.y = clientY - this.dragOffset.y;
    },
    stopDrag(e) {
      if (!this.isDragging) return;
      this.isDragging = false;
      
      document.removeEventListener('mousemove', this.onDrag);
      document.removeEventListener('touchmove', this.onDrag);
      document.removeEventListener('mouseup', this.stopDrag);
      document.removeEventListener('touchend', this.stopDrag);
      
      // Auto-collapse if dropped off-screen
      const windowWidth = window.innerWidth;
      const windowHeight = window.innerHeight;
      
      // Calculate center of the video window
      const centerX = this.windowPos.x + 120; // 240/2
      const centerY = this.windowPos.y + 90;  // 180/2
      
      const isOffScreen = 
        centerX < 0 || 
        centerX > windowWidth || 
        centerY < 0 || 
        centerY > windowHeight;
        
      if (isOffScreen && !this.isCollapsed) {
        this.isCollapsed = true;
        // Snap to nearest edge
        if (centerX < windowWidth / 2) {
          this.windowPos.x = 20;
        } else {
          this.windowPos.x = windowWidth - 68; // 48 + 20 margin
        }
        // Keep Y within bounds
        const safeMargin = 20;
        this.windowPos.y = Math.min(Math.max(safeMargin, this.windowPos.y), windowHeight - safeMargin - 48);
      } else {
        // Keep within bounds if not collapsing
        const safeMargin = 20;
        this.windowPos.x = Math.min(Math.max(-240 + safeMargin, this.windowPos.x), windowWidth - safeMargin);
        this.windowPos.y = Math.min(Math.max(0, this.windowPos.y), windowHeight - safeMargin);
      }
    },
    handleWindowClick() {
      console.log(7777)
      if (this.hasMoved && (Date.now() - this.dragStartTime > 200)) {
        // It was a drag, not a click
        return;
      }
      
      if (this.isCollapsed) {
        this.isCollapsed = false;
        // Move to safe position if needed
        const windowWidth = window.innerWidth;
        if (this.windowPos.x <= 0) this.windowPos.x = 20;
        else if (this.windowPos.x >= windowWidth - 48) this.windowPos.x = windowWidth - 260;
      }
    },
  },
  mounted() {
    // Initialize video window position to top-right
    this.windowPos = {
      x: window.innerWidth - 260, // 240 width + 20 margin
      y: 70
    };
    
    // Handle resize to keep window on screen
    window.addEventListener('resize', this.handleResize);

    // 初始化检查
    if (this.roomID === "" || !this.roomID) {
      this.$router.replace("/");
      return;
    }

    const that = this;

    if (this.isClient) {
      // 接收者端逻辑
      const dataPeer = this.shared.peers.local.data;

      // 确保peer已经就绪后再连接
      if (dataPeer.open) {
        this.connectToPeer();
      } else {
        dataPeer.on("open", () => {
          this.connectToPeer();
        });
      }
    } else {
      this.copyShareLink(true);
      const comm = new Comm();

      // 处理连接请求
      this.shared.peers.local.data.on("connection", (conn) => {
        conn.on("open", function () {
          msg.i(`Received connection from ${conn.peer}`);
          that.shared.peers.remote.data = conn;
          // 启动RTT定时测量（点对点模式）
          if (that.shared.app.method == 0) {
            if (that.rttTimer) clearInterval(that.rttTimer);
            const rttInterval =
              (import.meta.env.VITE_LATENCY_MEASUREMENT_INTERVAL_SECONDS
                ? parseFloat(import.meta.env.VITE_LATENCY_MEASUREMENT_INTERVAL_SECONDS)
                : 2) * 1e3;
            that.rttTimer = setInterval(() => {
              conn.send(comm.host.rttPing(Date.now()));
              that._rttPingSent = Date.now();
            }, rttInterval);
          }
          // Setup audio call listener (will be used when voice is enabled)
          that.setupAudioCallListener();
        });

        // 数据接收处理
        conn.on("data", function (data) {
          const commMsg = comm.resolve(data.toString());
          const status = commMsg.command;
          const peerID = commMsg.data ? commMsg.data.gid : undefined;
          switch (status) {
            case "connected": {
              msg.i(`Connection established with ${conn.peer}`);
              that.isReady = true;
              const video = that.getVideoElement();

              if (that.shared.app.method == 0) {
                // P2P 模式：捕获视频流并发送
                let videoEnded = false;

                const captureAndCall = () => {
                  if (!video) return;
                  let stream;
                  try {
                    stream = video.captureStream();
                  } catch (e) {
                    stream = video.mozCaptureStream();
                    msg.w(`Error: ${e} Attempting mozCaptureStream()...`);
                  }

                  // Apply track constraints (resolution/framerate) to captured stream
                  const videoTrack = stream.getVideoTracks()[0];
                  if (videoTrack) {
                    applyTrackConstraints(videoTrack, that.shared.app.videoQuality)
                      .then((ok) => ok && msg.i("Track constraints applied"))
                      .catch(() => { });
                  }

                  // Build call options with SDP bandwidth transform
                  const sdpTransform = createSdpTransform(that.shared.app.videoQuality);
                  const callOpts = sdpTransform ? { sdpTransform } : {};
                  const call = that.shared.peers.local.video.call(`${peerID}-video`, stream, callOpts);
                  that.shared.peers.remote.videoCall = call;

                  // Apply bitrate limit via RTCRtpSender once connection is active
                  if (call?.peerConnection) {
                    call.peerConnection.addEventListener("connectionstatechange", () => {
                      if (call.peerConnection.connectionState === "connected") {
                        applyBitrateLimit(call.peerConnection, that.shared.app.videoQuality)
                          .then((ok) => ok && msg.i("Bitrate limit applied"))
                          .catch(() => { });
                      }
                    });
                  }

                  msg.i("Video stream call initiated");
                };

                // 初始捕获并发送流
                captureAndCall();

                // 监听视频结束事件
                video?.addEventListener("ended", () => {
                  videoEnded = true;
                  msg.i("Host video ended, stream tracks may have ended");
                });

                // 当视频从结束状态恢复播放时，重新捕获流
                video?.addEventListener("playing", () => {
                  if (videoEnded) {
                    msg.i("Host video resumed from ended state, re-capturing stream");
                    videoEnded = false;
                    // 关闭旧的 call
                    if (that.shared.peers.remote.videoCall) {
                      that.shared.peers.remote.videoCall.close();
                    }
                    // 重新捕获并发起新的 call
                    captureAndCall();
                  }
                });

                that.shared.app.pingThread = setInterval(
                  () => {
                    conn.send(comm.host.timestamp());
                  },
                  import.meta.env.VITE_SAME_ORIGIN_SYNC_INTERVAL_SECONDS * 1e3,
                );
              } else {
                // 同源模式：发送视频 URL 并同步播放控制
                conn.send(comm.host.origin(that.shared.app.videoURL));
                video?.addEventListener("play", () => {
                  conn.send(comm.host.play());
                });
                video?.addEventListener("pause", () => {
                  conn.send(comm.host.pause());
                });
                video?.addEventListener("seeking", () => {
                  if (!video) return;
                  conn.send(comm.host.seek(video.currentTime));
                });
              }
              break;
            }
            case "progress": {
              const video = that.getVideoElement();
              const time = parseFloat(commMsg.data.cur);
              const atu = parseFloat(commMsg.data.atu);
              const offset = (Date.now() - atu) / 1e3;
              const delta = video ? video.currentTime - (time + offset) : 0;
              if (Math.abs(delta) > (import.meta.env.VITE_MAX_ACCEPTABLE_DELAY_SECONDS ?? 3)) {
                msg.w(`Attempting to force sync due to too much latency: ${delta}`);
                that.playbackDelta = delta;
                if (video) conn.send(comm.host.seek(video.currentTime + offset));
              } else {
                msg.i(`Received status report from the peer. Delta: ${delta}`);
                that.playbackDelta = delta;
              }
              conn.send(comm.host.latency(delta));
              break;
            }
            case "latency": {
              that.playbackDelta = commMsg.data.lat;
              break;
            }
            case "rtt-ping":
              // 对方发起RTT测量，立即回复pong
              conn.send(comm.host.rttPong(commMsg.data.ts));
              break;
            case "rtt-pong":
              // 收到pong，计算RTT
              if (commMsg.data.ts && commMsg.data.ts2) {
                const sent = parseInt(commMsg.data.ts);
                const now = Date.now();
                // RTT = (now - sent) - (ts2 - sent)
                that.rtt = (now - sent) / 1000;
              }
              break;
            case "voice-enabled":
              msg.i("Remote peer enabled voice call");
              that.remoteVoiceEnabled = true;
              // Setup listener to receive incoming audio calls
              that.setupAudioCallListener();
              // If we also have voice enabled, initiate the call
              if (that.shared.app.isVoiceEnabled && that.shared.app.audioStream) {
                that.startAudioCall();
              }
              break;
            case "voice-disabled":
              msg.i("Remote peer disabled voice call");
              that.remoteVoiceEnabled = false;
              break;
            case "shutdown":
              msg.w("Remote peer shut down the session");
              mduiAlert({
                headline: that.$t("StreamView.messages.shutdown.headline"),
                description: that.$t("StreamView.messages.shutdown.description"),
                confirmText: that.$t("StreamView.messages.shutdown.confirmText"),
                onConfirm: () => {
                  that.$router.replace("/");
                },
              });
              break;
            default: {
              msg.e(`Invalid message: ${data}`);
            }
          }
        });

        // 连接关闭处理
        conn.on("close", () => {
          msg.w("Connection closed");
          that.shared.peers.remote.data = null;
          that.isReady = false;
          that.remoteVoiceEnabled = false;
          const video = that.getVideoElement();
          video?.pause();
          if (that.rttTimer) clearInterval(that.rttTimer);
          that.rtt = null;
          // Clean up voice call on disconnect
          that.stopAudioCall();
        });
      });

      const videoEl = this.getVideoElement();
      if (videoEl) {
        if (this.shared.app.screenStream) {
          videoEl.srcObject = this.shared.app.screenStream;
        } else {videoEl.src = this.shared.app.videoURL;}
        videoEl.load();
      }
    }
    },
  beforeUnmount() {
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    window.removeEventListener('resize', this.handleResize);
    if (this.shared.app.syncThread) clearInterval(this.shared.app.syncThread);
    if (this.shared.app.pingThread) clearInterval(this.shared.app.pingThread);
    if (this.rttTimer) clearInterval(this.rttTimer);

    // Clean up voice call resources
    this.stopAudioCall();

    // Reset audio call listener state
    this.audioCallListenerRegistered = false;
    this.remoteVoiceEnabled = false;
  },

};
</script>

<style scoped>
.video-call-window {
  position: fixed;
  top: 0;
  left: 0;
  /* Removed fixed top/right/width/height here as they are controlled by style binding */
  width: 240px;
  height: 180px;
  background-color: #000;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  z-index: 1000;
  /* Transition handled in computed style */
  touch-action: none; /* Prevent browser handling of gestures */
}

.video-call-window.collapsed {
  background-color: rgb(var(--mdui-color-primary));
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
}

.collapsed-content {
  color: rgb(var(--mdui-color-on-primary));
  display: flex;
  align-items: center;
  justify-content: center;
}

.remote-call-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.local-call-video {
  position: absolute;
  bottom: 8px;
  right: 8px;
  width: 80px;
  height: 60px;
  background-color: #333;
  border-radius: 8px;
  object-fit: cover;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  z-index: 1001;
}

/* Mobile responsive adjustments */
@media (max-width: 600px) {
  .video-call-window {
    width: 160px;
    height: 120px;
    bottom: 80px; /* Make space for bottom controls if any */
  }
  
  .local-call-video {
    width: 50px;
    height: 37.5px;
  }
}
.stream-container {
  display: flex;
  flex-direction: column;
  padding: 2rem 2rem 80px 2rem;
  /* 增加底部内边距 */
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
  gap: 2rem;
  flex: 1;
}

.header-section {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  flex-wrap: wrap;
  gap: 1rem;
}

.title-group h1 {
  margin: 0;
  font-size: 2rem;
  font-weight: 500;
}

#status {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.9rem;
  color: rgb(var(--mdui-color-on-surface-variant));
  margin-top: 0.25rem;
}

.status-dot {
  font-size: 0.5rem;
  display: inline-flex;
  align-items: center;
  line-height: 1;
}

.voice-controls {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.5rem 1rem;
  background: rgb(var(--mdui-color-surface-container-low));
  border-radius: 12px;
}

.video-card {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 9;
  background-color: #000;
  border-radius: 24px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

#video-player-stream {
  width: 100%;
  height: 100%;
}

.loading-overlay {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: white;
  gap: 1rem;
}

.info-section {
  display: grid;
  /* grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); */
  grid-template-columns: 1fr 150px 1fr; 
  gap: 1.5rem;
}

.info-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 1.25rem;
}

.label {
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: rgb(var(--mdui-color-on-surface-variant));
  font-weight: 600;
}

.value {
  font-size: 1rem;
  color: rgb(var(--mdui-color-on-surface));
  word-break: break-all;
}

.latency {
  display: block;
  color: rgb(var(--mdui-color-on-surface-variant));
  font-size: 0.85rem;
  margin-top: 0.25rem;
}

.error-text {
  color: rgb(var(--mdui-color-error));
  margin: 0;
}

.hint-text {
  color: rgb(var(--mdui-color-on-surface-variant));
  font-size: 0.9rem;
  margin: 0;
}

.remote-audio {
  display: none;
}

.quality-card {
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.quality-buttons {
  margin-top: 0.25rem;
}

.quality-description {
  font-size: 0.8rem;
  color: rgb(var(--mdui-color-on-surface-variant));
  margin-top: 0.25rem;
  opacity: 0.85;
}

@media (max-width: 600px) {
  .stream-container {
    padding: 0;
    gap: 0;
  }

  .header-section {
    padding: 1.5rem 1rem;
  }

  .video-card {
    border-radius: 0;
    border: none;
  }

  .info-section {
    padding: 1rem;
    grid-template-columns: 1fr;
    gap: 1rem;
  }

  .info-group {
    border-radius: 12px;
  }
}
</style>
