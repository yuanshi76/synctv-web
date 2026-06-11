<script setup lang="ts">
import {
  computed,
  onBeforeUnmount,
  onMounted,
  ref,
  watch,
  defineAsyncComponent,
  nextTick
} from "vue";
import type { WatchStopHandle } from "vue";
import { useWebSocket, useResizeObserver } from "@vueuse/core";
import { useRouteParams } from "@vueuse/router";
import { roomStore } from "@/stores/room";
import { ElNotification, ElMessage } from "element-plus";
import { useMovieApi } from "@/hooks/useMovie";
import { useRoomApi, useRoomPermission } from "@/hooks/useRoom";
import artplayerPluginDanmuku from "artplayer-plugin-danmuku";
import { strLengthLimit, blobToUint8Array, formatTime } from "@/utils";
import { MessageType, Message, Status, messageTypeToJSON } from "@/proto/message";
import type { options } from "@/components/Player.vue";
import RoomInfo from "@/components/cinema/RoomInfo.vue";
import MovieList from "@/components/cinema/MovieList.vue";
import MoviePush from "@/components/cinema/MoviePush.vue";
import { RoomMemberPermission } from "@/types/Room";
import artplayerPluginAss from "@/plugins/artplayer-plugin-ass";
import { newSyncPlugin } from "@/plugins/sync";
import artplayerPluginMediaControl from "@/plugins/control";
import { artplayPluginSource } from "@/plugins/source";
import { currentMovieApi } from "@/services/apis/movie";
import { userStore } from "@/stores/user";
import { roomInfoApi } from "@/services/apis/room";
import { artplayerSubtitle } from "@/plugins/subtitle";
import { sendDanmu, artplayerStreamDanmu } from "@/plugins/danmu";
import { indexStore } from "@/stores";

const { settings } = indexStore();

const Player = defineAsyncComponent(() => import("@/components/Player.vue"));

const { token } = userStore();
// 获取房间信息
const room = roomStore();
const roomID = useRouteParams<string>("roomId");

const watchers: WatchStopHandle[] = [];
onBeforeUnmount(() => {
  watchers.forEach((w) => w());
});

const { getMovies, getCurrentMovie, isLoadingCurrent } = useMovieApi(token.value, roomID.value);
const { getMyInfo, myInfo } = useRoomApi();
const { state: roomInfo, execute: reqRoomInfoApi } = roomInfoApi();
const { hasMemberPermission } = useRoomPermission();

let player: Artplayer | undefined;

const wsProtocol = location.protocol === "https:" ? "wss:" : "ws:";
const { status, data, send, open } = useWebSocket(
  `${wsProtocol}//${window.location.host}/api/room/ws?roomId=${roomID.value}`,
  {
    ...(token.value ? { protocols: [token.value] } : {}),
    autoReconnect: {
      retries: 3,
      delay: 1000,
      onFailed() {
        ElMessage.error("Websocket 自动重连失败！");
      }
    },
    autoClose: true,
    immediate: false
  }
);

const sendElement = (msg: Message) => {
  if (!msg.timestamp) {
    msg.timestamp = Date.now();
  }
  console.groupCollapsed("Ws Send");
  console.log(messageTypeToJSON(msg.type));
  console.log(msg);
  console.groupEnd();
  return send(Message.encode(msg).finish() as any, false);
};

// 消息列表
const chatMsgList = ref<string[]>([]);
const sendChatText = (msg: string, onSuccess?: () => any, onFailed?: () => any) => {
  if (msg.length === 0) {
    ElMessage({
      message: "发送的消息不能为空",
      type: "warning"
    });
    if (onFailed) onFailed();
    return;
  }

  strLengthLimit(msg, 4096);
  sendElement(
    Message.create({
      type: MessageType.CHAT,
      chatContent: msg
    })
  );
  if (onSuccess) onSuccess();
};

const sendChatMsg_ = ref("");
const onSendSuccess = () => {
  sendChatMsg_.value = "";
};

const MAX_MESSAGE_COUNT = 64; // 设定聊天记录的最大长度
const sendMsg = (msg: string) => {
  chatMsgList.value.push(msg);
  // 如果超过聊天记录最大长度，则从前面开始删除多余的消息
  nextTick(() => {
    if (chatMsgList.value.length > MAX_MESSAGE_COUNT) {
      chatMsgList.value.splice(0, chatMsgList.value.length - MAX_MESSAGE_COUNT);
    }
    // 将新消息存储到 sessionStorage
    sessionStorage.setItem(`chatMessages-${roomID}`, JSON.stringify(chatMsgList.value));
  });

  // 确保聊天区域滚动到底部
  nextTick(() => {
    if (chatArea.value) chatArea.value.scrollTop = chatArea.value.scrollHeight;
  });
};

const playerOption = computed<options>(() => {
  if (!room.currentMovie.base!.url) {
    return {
      url: ""
    };
  }
  let option: options = {
    url: room.currentMovie.base!.url,
    type: room.currentMovie.base!.type,
    isLive: room.currentMovie.base!.live,
    headers: room.currentMovie.base!.headers,
    p2pZone: settings?.p2pZone,
    plugins: [
      // 弹幕
      artplayerPluginDanmuku({
        danmuku: room.currentMovie.base!.danmu || [],
        speed: 8,
        async beforeEmit(danmu: any) {
          if (danmu.direct) {
            return true;
          }
          sendChatText(danmu.text);
          return false;
        }
      }),
      // WARN: room.currentStatus 变了会导致重载
      newSyncPlugin(sendElement, room.currentStatus, () => room.currentExpireId),
      artplayerPluginMediaControl(),
    ]
  };

  if (room.currentMovie.base!.moreSources) {
    const obj = room.currentMovie.base!.moreSources || [];
    option.plugins!.push(
      artplayPluginSource([
        {
          url: option.url,
          html: "默认",
          type: option.type || ""
        },
        ...obj.map((item) => ({
          url: item.url,
          html: item.name,
          type: item.type
        }))
      ])
    );
  }

  if (room.currentMovie.base!.subtitles) {
    let defaultUrl;
    let useAssPlugin = false;

    // deep copy
    const subtitle = Object.assign({}, room.currentMovie.base!.subtitles);

    for (let key in subtitle) {
      if (subtitle[key].type === "ass") {
        useAssPlugin = true;
        defaultUrl = subtitle[key].url;
        break;
      }
    }

    option.plugins!.push(artplayerSubtitle(subtitle));
    // return;
    useAssPlugin &&
      option.plugins!.push(
        artplayerPluginAss({
          // debug: true,
          subUrl: defaultUrl
        })
      );
  }

  if (room.currentMovie.base!.streamDanmu) {
    option.plugins!.push(artplayerStreamDanmu(room.currentMovie.base!.streamDanmu));
  }

  return option;
});

const { state: currentMovie, execute: reqCurrentMovieApi } = currentMovieApi();
const updateSources = async () => {
  try {
    await reqCurrentMovieApi({
      headers: { Authorization: token.value, "X-Room-Id": roomID.value }
    });
    if (!currentMovie.value) return;
    if (currentMovie.value.movie.base.url.startsWith("/")) {
      currentMovie.value.movie.base.url = `${window.location.origin}${currentMovie.value.movie.base.url}`;
    }
    if (
      currentMovie.value.movie.base.moreSources &&
      currentMovie.value.movie.base.moreSources.length > 0
    ) {
      for (let i = 0; i < currentMovie.value.movie.base.moreSources.length; i++) {
        if (currentMovie.value.movie.base.moreSources[i].url.startsWith("/")) {
          currentMovie.value.movie.base.moreSources[i].url =
            `${window.location.origin}${currentMovie.value.movie.base.moreSources[i].url}`;
        }
      }
    }
    if (!player) return;
    room.currentExpireId = currentMovie.value.expireId;
    const moreSources = currentMovie.value.movie.base.moreSources || [];
    player.plugins["source"].updateSources([
      {
        url: currentMovie.value.movie.base.url,
        html: "默认",
        type: currentMovie.value.movie.base.type || ""
      },
      ...moreSources.map((item) => ({
        url: item.url,
        html: item.name,
        type: item.type
      }))
    ]);
  } catch (err: any) {
    console.log(err);
    ElNotification({
      title: "获取影片列表失败",
      message: err.response.data.error || err.message,
      type: "error"
    });
  }
};

const getPlayerInstance = (art: Artplayer) => {
  player = art;
  listenPlayerType(player);
};

const playType = ref<string | undefined>();

const listenPlayerType = (player: Artplayer) => {
  player.once("ready", () => {
    playType.value = player?.option.type;
    player.on("restart", () => {
      playType.value = player?.option.type;
    });
    player.on("destroy", () => {
      playType.value = undefined;
    });
  });
};

const setPlayerStatus = (status: Status) => {
  if (!player) return;
  player.plugins["syncPlugin"].setAndNoPublishStatus(status);
};

// key: userId:connId
const peerConnections = ref<{ [key: string]: RTCPeerConnection }>({});
// Perfect-negotiation state per peer, keyed by userId:connId
const peerStates: {
  [key: string]: { polite: boolean; makingOffer: boolean; ignoreOffer: boolean };
} = {};
const localStream = ref<MediaStream | undefined>(undefined);
let remoteAudioElements: { [key: string]: HTMLAudioElement } = {};

const defaultICEServers: RTCIceServer[] = [{ urls: "stun:stun.l.google.com:19302" }];

// Build the ICE server list from the public settings, falling back to a
// built-in STUN server when unset or malformed.
const getICEServers = (): RTCIceServer[] => {
  const raw = settings?.webrtcICEServers;
  if (raw) {
    try {
      const parsed = JSON.parse(raw);
      if (Array.isArray(parsed) && parsed.length > 0) {
        return parsed as RTCIceServer[];
      }
    } catch (err) {
      console.error("解析 webrtcICEServers 失败，使用默认 STUN:", err);
    }
  }
  return defaultICEServers;
};

const peerConnectionsLengthWithUserId = computed(() => {
  const userIdSet = new Set(Object.keys(peerConnections.value).map((key) => key.split(":")[0]));
  return userIdSet.size;
});

// 音频设备列表
const audioInputDevices = ref<MediaDeviceInfo[]>([]);
const audioOutputDevices = ref<MediaDeviceInfo[]>([]);
const selectedAudioInput = ref("");
const selectedAudioOutput = ref("");

// 视频(摄像头)设备列表与状态
const videoInputDevices = ref<MediaDeviceInfo[]>([]);
const selectedVideoInput = ref("");
const isCameraOn = ref(false);
// 摄像头视频流单独保存，避免 switchMicrophone 误停视频轨道
const localVideoStream = ref<MediaStream | undefined>(undefined);
// 远端视频流，key: userId:connId
const remoteVideoStreams = ref<{ [key: string]: MediaStream }>({});
// 视频发送码率上限(bps)，限制 mesh 拓扑下的上行带宽
const videoMaxBitrate = 600_000;
const screenMaxBitrate = 1_500_000;

// 屏幕共享：本地共享流、是否正在共享、远端屏幕流，以及各对端声明的屏幕流 id
const localScreenStream = ref<MediaStream | undefined>(undefined);
const isScreenSharing = ref(false);
const remoteScreenStreams = ref<{ [key: string]: MediaStream }>({});
const peerScreenStreamId: { [key: string]: string | undefined } = {};
// mesh 下人人互联：只要有任一对端在推屏幕流，即视为"已有人在共享"
const someoneElseSharing = computed(() => Object.keys(remoteScreenStreams.value).length > 0);

// 将 MediaStream 绑定到 <video>/<audio> 元素的 srcObject
const vSrcobject = {
  mounted(el: HTMLMediaElement, binding: { value: MediaStream | undefined }) {
    el.srcObject = binding.value ?? null;
  },
  updated(el: HTMLMediaElement, binding: { value: MediaStream | undefined }) {
    if (el.srcObject !== binding.value) {
      el.srcObject = binding.value ?? null;
    }
  }
};

const outputVolume = ref(1.0); // 扬声器音量
const isMuted = ref(false); // 麦克风静音状态

// 获取音视频设备列表
const getAudioDevices = async () => {
  const devices = await navigator.mediaDevices.enumerateDevices();
  audioInputDevices.value = devices.filter((device) => device.kind === "audioinput");
  audioOutputDevices.value = devices.filter((device) => device.kind === "audiooutput");
  videoInputDevices.value = devices.filter((device) => device.kind === "videoinput");
};

// 切换麦克风静音状态
const toggleMute = () => {
  if (!localStream.value) return;
  isMuted.value = !isMuted.value;
  localStream.value.getAudioTracks().forEach((track) => {
    track.enabled = !isMuted.value;
  });
};

// 切换麦克风
const switchMicrophone = async () => {
  if (!localStream.value) return;

  try {
    const newStream = await navigator.mediaDevices.getUserMedia({
      audio: {
        deviceId: selectedAudioInput.value,
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true
      }
    });

    // 停止旧轨道
    localStream.value.getTracks().forEach((track) => track.stop());

    // 替换所有PeerConnection中的轨道
    const [audioTrack] = newStream.getTracks();
    audioTrack.enabled = !isMuted.value; // 保持当前的静音状态

    for (const pc of Object.values(peerConnections.value)) {
      const sender = pc.getSenders().find((s: any) => s.track?.kind === "audio");
      if (sender) {
        await sender.replaceTrack(audioTrack);
      }
    }

    localStream.value = newStream;
  } catch (err) {
    ElMessage.error(`切换麦克风失败: ${err}`);
  }
};

// 切换扬声器
const switchSpeaker = async () => {
  try {
    for (const audio of Object.values(remoteAudioElements)) {
      if ("setSinkId" in audio) {
        await (audio as any).setSinkId(selectedAudioOutput.value);
        audio.volume = outputVolume.value;
      }
    }
  } catch (err) {
    ElMessage.error(`切换扬声器失败: ${err}`);
  }
};

// 调整扬声器音量
const adjustOutputVolume = () => {
  Object.values(remoteAudioElements).forEach((audio) => {
    audio.volume = outputVolume.value;
  });
};

// 限制视频发送码率，避免 mesh 拓扑下上行带宽随人数线性膨胀。
// degradationPreference 在带宽受限时决定"降帧率"还是"降分辨率"：
// 屏幕共享多为文字，应保分辨率(maintain-resolution)以保证可读性。
const applyVideoSenderParams = async (
  sender: RTCRtpSender,
  maxBitrate = videoMaxBitrate,
  degradationPreference?: "maintain-framerate" | "maintain-resolution" | "balanced"
) => {
  try {
    const params = sender.getParameters();
    if (!params.encodings || params.encodings.length === 0) {
      params.encodings = [{}];
    }
    params.encodings[0].maxBitrate = maxBitrate;
    if (degradationPreference) {
      // degradationPreference 在部分 TS lib.dom 中尚未声明，做局部断言
      (params as RTCRtpSendParameters & { degradationPreference?: string }).degradationPreference =
        degradationPreference;
    }
    await sender.setParameters(params);
  } catch (err) {
    console.error("设置视频发送参数失败:", err);
  }
};

// 切换摄像头开关
const toggleCamera = async () => {
  if (!localStream.value) return;
  if (isCameraOn.value) {
    await stopCamera();
  } else {
    await startCamera();
  }
};

const startCamera = async () => {
  try {
    localVideoStream.value = await navigator.mediaDevices.getUserMedia({
      video: selectedVideoInput.value ? { deviceId: selectedVideoInput.value } : true
    });
  } catch (err) {
    ElMessage.error(`开启摄像头失败！${err}`);
    return;
  }
  const [videoTrack] = localVideoStream.value.getVideoTracks();
  if (!videoTrack) return;
  // 摄像头画面以动态为主，提示编码器优先保帧率
  videoTrack.contentHint = "motion";
  // 向所有已建立的连接添加视频轨道，触发 onnegotiationneeded 重协商
  for (const pc of Object.values(peerConnections.value)) {
    const sender = pc.addTrack(videoTrack, localVideoStream.value);
    await applyVideoSenderParams(sender);
  }
  isCameraOn.value = true;
};

const stopCamera = async () => {
  for (const pc of Object.values(peerConnections.value)) {
    for (const sender of pc.getSenders()) {
      if (sender.track?.kind === "video") {
        pc.removeTrack(sender);
      }
    }
  }
  localVideoStream.value?.getTracks().forEach((track) => track.stop());
  localVideoStream.value = undefined;
  isCameraOn.value = false;
};

// 摄像头开启状态下切换设备
const switchCamera = async () => {
  if (!isCameraOn.value || !localVideoStream.value) return;
  let newStream: MediaStream;
  try {
    newStream = await navigator.mediaDevices.getUserMedia({
      video: selectedVideoInput.value ? { deviceId: selectedVideoInput.value } : true
    });
  } catch (err) {
    ElMessage.error(`切换摄像头失败: ${err}`);
    return;
  }
  const [videoTrack] = newStream.getVideoTracks();
  for (const pc of Object.values(peerConnections.value)) {
    const sender = pc.getSenders().find((s) => s.track?.kind === "video");
    if (sender) {
      await sender.replaceTrack(videoTrack);
    }
  }
  localVideoStream.value.getTracks().forEach((track) => track.stop());
  localVideoStream.value = newStream;
};

// 切换屏幕共享
const toggleScreenShare = async () => {
  if (!localStream.value) return;
  if (isScreenSharing.value) {
    await stopScreenShare();
  } else {
    await startScreenShare();
  }
};

const startScreenShare = async () => {
  if (someoneElseSharing.value) {
    ElMessage.warning("已有成员正在共享屏幕");
    return;
  }
  try {
    // 采集屏幕视频与(可选的)系统/标签页音频；显示光标、避免选中本应用标签页、
    // 允许中途切换共享源。部分约束在 TS lib.dom 中尚未声明，整体做一次断言。
    localScreenStream.value = await navigator.mediaDevices.getDisplayMedia({
      video: {
        cursor: "always",
        frameRate: { ideal: 15, max: 30 },
        width: { ideal: 1920 },
        height: { ideal: 1080 }
      },
      audio: true,
      systemAudio: "include",
      selfBrowserSurface: "exclude",
      surfaceSwitching: "include"
    } as MediaStreamConstraints);
  } catch (err) {
    // 用户取消选择时也会进入此分支，不弹错误
    return;
  }
  const [screenTrack] = localScreenStream.value.getVideoTracks();
  if (!screenTrack) return;
  // 屏幕内容多为文字，提示编码器优先保分辨率(可读性)
  screenTrack.contentHint = "text";
  // 响应浏览器原生"停止共享"按钮
  screenTrack.onended = () => {
    stopScreenShare();
  };
  // 添加屏幕的全部轨道(视频 + 系统音频)，触发重协商
  for (const pc of Object.values(peerConnections.value)) {
    for (const track of localScreenStream.value.getTracks()) {
      const sender = pc.addTrack(track, localScreenStream.value);
      if (track.kind === "video") {
        await applyVideoSenderParams(sender, screenMaxBitrate, "maintain-resolution");
      }
    }
  }
  isScreenSharing.value = true;
};

const stopScreenShare = async () => {
  const screenTracks = new Set(localScreenStream.value?.getTracks() ?? []);
  for (const pc of Object.values(peerConnections.value)) {
    for (const sender of pc.getSenders()) {
      if (sender.track && screenTracks.has(sender.track)) {
        pc.removeTrack(sender);
      }
    }
  }
  localScreenStream.value?.getTracks().forEach((track) => track.stop());
  localScreenStream.value = undefined;
  isScreenSharing.value = false;
};

// 双击视频块在全屏与还原之间切换，便于专注观看共享的屏幕。
const toggleFullscreen = (event: MouseEvent) => {
  const el = event.currentTarget as HTMLVideoElement | null;
  if (!el) return;
  if (document.fullscreenElement) {
    document.exitFullscreen().catch((err) => console.error("退出全屏失败:", err));
  } else {
    el.requestFullscreen().catch((err) => console.error("进入全屏失败:", err));
  }
};

const joinWebRTC = async () => {
  try {
    await getAudioDevices();
    localStream.value = await navigator.mediaDevices.getUserMedia({
      audio: selectedAudioInput.value ? { deviceId: selectedAudioInput.value } : true
    });
  } catch (err) {
    ElMessage.error(`获取媒体流失败！${err}`);
    return;
  }
  await sendElement(
    Message.create({
      type: MessageType.WEBRTC_JOIN
    })
  );
};

const exitWebRTC = async () => {
  await sendElement(
    Message.create({
      type: MessageType.WEBRTC_LEAVE
    })
  );
  for (const id in peerConnections.value) {
    const pc = peerConnections.value[id];
    pc.close();
    delete peerConnections.value[id];
    delete peerStates[id];
  }
  localStream.value!.getTracks().forEach((track) => track.stop());
  localStream.value = undefined;
  localVideoStream.value?.getTracks().forEach((track) => track.stop());
  localVideoStream.value = undefined;
  isCameraOn.value = false;
  remoteVideoStreams.value = {};
  localScreenStream.value?.getTracks().forEach((track) => track.stop());
  localScreenStream.value = undefined;
  isScreenSharing.value = false;
  remoteScreenStreams.value = {};
};

const handleWebrtcJoin = async (msg: Message) => {
  // Existing peers initiate the connection toward the joining peer and take the
  // impolite role. Adding the local tracks fires onnegotiationneeded, which
  // sends the offer automatically.
  createPeerConnection(msg.webrtcData!.from, false);
};

const handleWebrtcLeave = async (msg: Message) => {
  closePeerConnection(msg.webrtcData!.from);
};

const closePeerConnection = (id: string) => {
  const pc = peerConnections.value[id];
  if (pc) {
    pc.close();
    delete peerConnections.value[id];
  }
  delete peerStates[id];
  delete remoteVideoStreams.value[id];
  delete remoteScreenStreams.value[id];
  delete peerScreenStreamId[id];
  // 同时清理该对端的麦克风音频与屏幕系统音频元素。
  for (const key of [id, `screen:${id}`]) {
    const remoteAudio = remoteAudioElements[key];
    if (remoteAudio) {
      remoteAudio.pause();
      remoteAudio.srcObject = null;
      delete remoteAudioElements[key];
    }
  }
};

const handleWebrtcOffer = async (msg: Message) => {
  const id = msg.webrtcData!.from;
  const payload = JSON.parse(msg.webrtcData!.data);
  // Unwrap the envelope { sdp, screenId }; fall back to a raw description for
  // robustness. Record which remote stream id (if any) carries the screen.
  const description = (payload.sdp ?? payload) as RTCSessionDescriptionInit;
  peerScreenStreamId[id] = payload.screenId;
  let pc = peerConnections.value[id];
  if (!pc) {
    // First offer from this peer (we are the joining side) → polite responder.
    pc = createPeerConnection(id, true);
  }
  const state = peerStates[id];
  const offerCollision =
    description.type === "offer" && (state.makingOffer || pc.signalingState !== "stable");
  state.ignoreOffer = !state.polite && offerCollision;
  if (state.ignoreOffer) {
    return;
  }
  await pc.setRemoteDescription(new RTCSessionDescription(description));
  await pc.setLocalDescription();
  // Send the answer back to the offering peer.
  sendElement(
    Message.create({
      type: MessageType.WEBRTC_ANSWER,
      webrtcData: {
        data: JSON.stringify(pc.localDescription),
        to: id
      }
    })
  );
};

const createPeerConnection = (id: string, polite: boolean) => {
  const pc = new RTCPeerConnection({
    iceServers: getICEServers(),
    iceCandidatePoolSize: 10
  });
  peerStates[id] = { polite, makingOffer: false, ignoreOffer: false };

  // Perfect negotiation: let the browser decide when (re)negotiation is needed.
  pc.onnegotiationneeded = async () => {
    const state = peerStates[id];
    if (!state) return;
    try {
      state.makingOffer = true;
      await pc.setLocalDescription();
      sendElement(
        Message.create({
          type: MessageType.WEBRTC_OFFER,
          webrtcData: {
            // Envelope carries the screen-share stream id so receivers can tell
            // a screen track apart from a camera track. The backend treats
            // `data` as opaque, so no proto change is needed.
            data: JSON.stringify({
              sdp: pc.localDescription,
              screenId: localScreenStream.value?.id
            }),
            to: id
          }
        })
      );
    } catch (err) {
      console.error("onnegotiationneeded 失败:", err);
    } finally {
      state.makingOffer = false;
    }
  };

  pc.onicecandidate = (event) => {
    if (event.candidate) {
      sendElement(
        Message.create({
          type: MessageType.WEBRTC_ICE_CANDIDATE,
          webrtcData: {
            data: JSON.stringify(event.candidate),
            to: id
          }
        })
      );
    }
  };
  pc.ontrack = (event) => {
    const stream = event.streams[0];
    // 通过对端 offer 信封声明的 screenId 判断该流是屏幕流还是摄像头/麦克风流。
    const isScreen = !!peerScreenStreamId[id] && stream.id === peerScreenStreamId[id];
    if (event.track.kind === "video") {
      bindRemoteVideoTrack(id, event.track, stream, isScreen);
    } else {
      bindRemoteAudioTrack(id, event.track, stream, isScreen);
    }
  };

  // 音频轨道
  localStream.value!.getTracks().forEach((track) => pc.addTrack(track, localStream.value!));
  // 若本地摄像头已开启，向新连接补发视频轨道(晚加入者也能看到画面)
  if (localVideoStream.value) {
    for (const track of localVideoStream.value.getVideoTracks()) {
      const sender = pc.addTrack(track, localVideoStream.value);
      applyVideoSenderParams(sender);
    }
  }
  // 若本地正在共享屏幕，同样补发屏幕的全部轨道(视频 + 系统音频)
  if (localScreenStream.value) {
    for (const track of localScreenStream.value.getTracks()) {
      const sender = pc.addTrack(track, localScreenStream.value);
      if (track.kind === "video") {
        applyVideoSenderParams(sender, screenMaxBitrate, "maintain-resolution");
      }
    }
  }
  peerConnections.value[id] = pc;
  return pc;
};

// 绑定远端视频轨道到响应式 map，并处理静音/结束时的画面增删。
// isScreen 由调用方依据 offer 信封里的 screenId 判定，用于区分屏幕流与摄像头流。
const bindRemoteVideoTrack = (
  id: string,
  track: MediaStreamTrack,
  stream: MediaStream,
  isScreen: boolean
) => {
  const target = isScreen ? remoteScreenStreams : remoteVideoStreams;
  target.value[id] = stream;
  track.onunmute = () => {
    target.value[id] = stream;
  };
  track.onmute = () => {
    delete target.value[id];
  };
  track.onended = () => {
    delete target.value[id];
  };
};

// 绑定远端音频轨道并播放。屏幕的系统音频与麦克风音频使用不同的 key
// (screen:${id} vs id)，避免共享屏幕时覆盖对端的麦克风音频元素。
const bindRemoteAudioTrack = (
  id: string,
  track: MediaStreamTrack,
  stream: MediaStream,
  isScreen: boolean
) => {
  const key = isScreen ? `screen:${id}` : id;
  // 重协商时 ontrack 可能再次触发，已存在则复用元素仅更新 srcObject。
  const existing = remoteAudioElements[key];
  if (existing) {
    existing.srcObject = stream;
    return;
  }
  const remoteAudio = document.createElement("audio");
  remoteAudio.srcObject = stream;
  remoteAudio.volume = outputVolume.value;
  if (selectedAudioOutput.value && "setSinkId" in remoteAudio) {
    (remoteAudio as any).setSinkId(selectedAudioOutput.value).catch((error: any) => {
      console.error("扬声器设置失败:", error);
    });
  }
  remoteAudio.style.display = "none";
  track.onended = () => {
    remoteAudio.pause();
    remoteAudio.srcObject = null;
    delete remoteAudioElements[key];
  };
  remoteAudioElements[key] = remoteAudio;
  remoteAudio.play().catch((error) => {
    console.error("Audio playback failed:", error);
  });
};

const handleWebrtcAnswer = async (msg: Message) => {
  const description = JSON.parse(msg.webrtcData!.data) as RTCSessionDescriptionInit;
  const pc = peerConnections.value[msg.webrtcData!.from];
  if (!pc) return;
  await pc.setRemoteDescription(new RTCSessionDescription(description));
};

const handleWebrtcIceCandidate = async (msg: Message) => {
  const id = msg.webrtcData!.from;
  const candidate: RTCIceCandidateInit = JSON.parse(msg.webrtcData!.data);
  const pc = peerConnections.value[id];
  if (!pc) return;
  try {
    await pc.addIceCandidate(new RTCIceCandidate(candidate));
  } catch (err) {
    // A discarded offer (impolite peer during glare) also discards its
    // candidates; ignore the resulting errors instead of surfacing them.
    if (!peerStates[id]?.ignoreOffer) {
      throw err;
    }
  }
};

const handleElementMessage = (msg: Message) => {
  console.groupCollapsed("Ws Message");
  console.log(messageTypeToJSON(msg.type));
  console.info(msg);
  console.groupEnd();
  switch (msg.type) {
    case MessageType.ERROR: {
      console.error(msg.errorMessage);
      ElNotification({
        title: "错误",
        message: msg.errorMessage,
        type: "error"
      });
      break;
    }

    case MessageType.WEBRTC_JOIN: {
      handleWebrtcJoin(msg);
      break;
    }
    case MessageType.WEBRTC_LEAVE: {
      handleWebrtcLeave(msg);
      break;
    }
    case MessageType.WEBRTC_OFFER: {
      handleWebrtcOffer(msg);
      break;
    }
    case MessageType.WEBRTC_ANSWER: {
      handleWebrtcAnswer(msg);
      break;
    }
    case MessageType.WEBRTC_ICE_CANDIDATE: {
      handleWebrtcIceCandidate(msg);
      break;
    }

    // 聊天消息
    case MessageType.CHAT: {
      if (!msg.chatContent) {
        return;
      }
      const currentTime = formatTime(new Date()); // 格式化时间
      const senderName = msg.sender?.username;
      const messageContent = `${senderName}: ${msg.chatContent}`;
      const messageWithTime = `${messageContent} <small>[${currentTime}]</small>`;
      // 添加消息到消息列表
      sendMsg(messageWithTime);
      sendDanmu({ text: messageContent, border: true }, player);
      break;
    }
    case MessageType.STATUS:
    case MessageType.CHECK_STATUS:
    case MessageType.SYNC: {
      switch (msg.type) {
        case MessageType.CHECK_STATUS:
          ElNotification({
            title: "状态不同步，同步中...",
            type: "warning"
          });
          break;
        case MessageType.SYNC:
          ElNotification({
            title: "同步成功",
            type: "success"
          });
          break;
      }
      setPlayerStatus(msg.playbackStatus!);
      break;
    }

    case MessageType.EXPIRED: {
      ElNotification({
        title: "链接过期,刷新中",
        type: "info"
      });
      updateSources();
      break;
    }

    // 设置正在播放的影片
    case MessageType.CURRENT: {
      getCurrentMovie();
      break;
    }

    // 播放列表更新
    case MessageType.MOVIES: {
      // 如果有权限则获取
      if (can(RoomMemberPermission.PermissionGetMovieList)) getMovies();
      break;
    }

    case MessageType.VIEWER_COUNT: {
      room.viewerCount = msg.viewerCount!;
      break;
    }

    case MessageType.MY_STATUS: {
      try {
        getMyInfo(roomID.value);
      } catch (err: any) {
        console.error(err);
        ElNotification({
          title: "错误",
          message: err.response.data.error || err.message,
          type: "error"
        });
        return;
      }
      break;
    }
  }
};

const noPlayArea = ref();
const playArea = ref();

// 消息区域
const chatArea = ref();

// 设置聊天框高度
const resetChatAreaHeight = () => {
  const h = playArea.value ? playArea : noPlayArea;
  if (!chatArea.value || !h.value) return;

  // 计算基础高度
  let baseHeight = h.value.scrollHeight - 112;

  // 如果有语音控制面板,减去其高度
  if (audioControls.value && audioControls.value instanceof HTMLElement) {
    baseHeight -= audioControls.value.offsetHeight + 16; // 16px for margin
  }

  chatArea.value.style.height = `${baseHeight}px`;
};

const card = ref(null);
useResizeObserver(card, resetChatAreaHeight);

const audioControls = ref<HTMLElement | null>(null);
useResizeObserver(audioControls, resetChatAreaHeight);

const can = (p: RoomMemberPermission) => {
  if (!myInfo.value) return;
  const myP = myInfo.value.permissions;
  return hasMemberPermission(myP, p);
};

const p = async () => {
  if (can(RoomMemberPermission.PermissionGetMovieList)) await getMovies();
  await getCurrentMovie();
};

onMounted(async () => {
  // 获取房间信息
  try {
    await reqRoomInfoApi({
      headers: { Authorization: token.value, "X-Room-Id": roomID.value }
    });
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "错误",
      message: err.response.data.error || err.message,
      type: "error"
    });
    return;
  }

  // 获取用户信息
  try {
    await getMyInfo(roomID.value);
  } catch (err: any) {
    console.error(err);
    ElNotification({
      title: "错误",
      message: err.response.data.error || err.message,
      type: "error"
    });
    return;
  }

  // 从 sessionStorage 获取存储的聊天消息
  const storedMessages = sessionStorage.getItem(`chatMessages-${roomID}`);
  if (storedMessages) {
    chatMsgList.value = JSON.parse(storedMessages);
  }

  // 启动websocket连接
  open();

  // 监听ws信息变化
  watchers.push(
    watch(
      () => data.value,
      async () => {
        try {
          const arr = await blobToUint8Array(data.value);
          handleElementMessage(Message.decode(arr));
        } catch (err: any) {
          console.error(err);
          ElMessage.error(err.message);
        }
      }
    )
  );

  await p();

  // 获取初始音频设备列表
  await getAudioDevices();

  // 监听设备变化
  navigator.mediaDevices.addEventListener("devicechange", getAudioDevices);
});

onBeforeUnmount(() => {
  navigator.mediaDevices.removeEventListener("devicechange", getAudioDevices);
});
</script>

<template>
  <el-row :gutter="20">
    <el-col :md="18" class="mb-5 max-sm:my-2">
      <div class="card" ref="card">
        <div
          class="card-title flex flex-wrap justify-between max-sm:text-sm max-sm:pb-4"
          v-if="playerOption.url"
        >
          <el-tag v-if="playType">{{ playType }}</el-tag>
          {{ room.currentMovie.base!.name }}
          <small class="ml-2">👁‍🗨 {{ room.viewerCount }} </small>
        </div>
        <div class="card-title flex flex-wrap justify-between max-sm:text-sm" v-else>
          当前没有影片播放，快去添加几部吧~<small class="font-normal"
            >👁‍🗨 {{ room.viewerCount }}
          </small>
        </div>
        <div class="card-body max-sm:p-0 pb-4" ref="playArea" v-if="playerOption.url">
          <div class="art-player">
            <Player @get-instance="getPlayerInstance" :options="playerOption"></Player>
          </div>
        </div>
        <div class="card-body max-sm:pb-3 max-sm:px-3" ref="noPlayArea" v-else>
          <div v-if="isLoadingCurrent" class="flex justify-center items-center h-[550px]">
            <div class="loading-spinner">
              <div class="bounce1"></div>
              <div class="bounce2"></div>
              <div class="bounce3"></div>
            </div>
          </div>
          <img v-else class="mx-auto" src="/src/assets/something-lost.webp" />
        </div>
      </div>
    </el-col>
    <el-col :md="6" class="mb-5 max-sm:mb-2">
      <div class="card h-full">
        <div class="card-title">
          在线聊天
          <el-button
            v-if="!localStream && can(RoomMemberPermission.PermissionWebRTC)"
            @click="joinWebRTC"
            type="primary"
            size="small"
            style="float: right"
            >加入语音</el-button
          >
          <el-button
            v-else-if="localStream && can(RoomMemberPermission.PermissionWebRTC)"
            @click="exitWebRTC"
            type="primary"
            size="small"
            style="float: right"
            >退出语音 ({{ peerConnectionsLengthWithUserId }})</el-button
          >
        </div>
        <div
          ref="audioControls"
          v-show="localStream"
          class="card-body mb-2 audio-controls-container"
        >
          <div class="audio-controls">
            <el-select
              v-model="selectedAudioInput"
              placeholder="选择麦克风"
              @change="switchMicrophone"
            >
              <el-option
                v-for="device in audioInputDevices"
                :key="device.deviceId"
                :label="device.label || `麦克风 ${device.deviceId.slice(0, 8)}`"
                :value="device.deviceId"
              />
            </el-select>

            <el-select
              v-model="selectedAudioOutput"
              placeholder="选择扬声器"
              @change="switchSpeaker"
            >
              <el-option
                v-for="device in audioOutputDevices"
                :key="device.deviceId"
                :label="device.label || `扬声器 ${device.deviceId.slice(0, 8)}`"
                :value="device.deviceId"
              />
            </el-select>

            <div class="volume-control">
              <span>扬声器音量:</span>
              <el-slider
                v-model="outputVolume"
                :min="0"
                :max="1"
                :step="0.1"
                @change="adjustOutputVolume"
              />
            </div>

            <el-button
              @click="toggleMute"
              :type="isMuted ? 'danger' : 'primary'"
              size="small"
              style="width: 100%"
            >
              {{ isMuted ? "取消闭麦" : "闭麦" }}
            </el-button>

            <el-select
              v-if="videoInputDevices.length"
              v-model="selectedVideoInput"
              placeholder="选择摄像头"
              @change="switchCamera"
            >
              <el-option
                v-for="device in videoInputDevices"
                :key="device.deviceId"
                :label="device.label || `摄像头 ${device.deviceId.slice(0, 8)}`"
                :value="device.deviceId"
              />
            </el-select>

            <el-button
              @click="toggleCamera"
              :type="isCameraOn ? 'danger' : 'primary'"
              size="small"
              style="width: 100%"
            >
              {{ isCameraOn ? "关闭摄像头" : "开启摄像头" }}
            </el-button>

            <el-button
              @click="toggleScreenShare"
              :type="isScreenSharing ? 'danger' : 'primary'"
              :disabled="!isScreenSharing && someoneElseSharing"
              size="small"
              style="width: 100%"
            >
              {{
                isScreenSharing
                  ? "停止共享"
                  : someoneElseSharing
                    ? "他人共享中"
                    : "共享屏幕"
              }}
            </el-button>
          </div>
        </div>
        <div
          v-show="
            localStream &&
            (isCameraOn ||
              isScreenSharing ||
              Object.keys(remoteVideoStreams).length ||
              Object.keys(remoteScreenStreams).length)
          "
          class="card-body mb-2 video-wall"
        >
          <video
            v-if="isScreenSharing && localScreenStream"
            v-srcobject="localScreenStream"
            autoplay
            playsinline
            muted
            title="双击全屏"
            class="video-tile screen-tile"
            @dblclick="toggleFullscreen"
          ></video>
          <video
            v-for="(stream, id) in remoteScreenStreams"
            :key="`screen-${id}`"
            v-srcobject="stream"
            autoplay
            playsinline
            muted
            title="双击全屏"
            class="video-tile screen-tile"
            @dblclick="toggleFullscreen"
          ></video>
          <video
            v-if="isCameraOn && localVideoStream"
            v-srcobject="localVideoStream"
            autoplay
            playsinline
            muted
            class="video-tile"
          ></video>
          <video
            v-for="(stream, id) in remoteVideoStreams"
            :key="id"
            v-srcobject="stream"
            autoplay
            playsinline
            muted
            class="video-tile"
          ></video>
        </div>
        <div class="card-body mb-2">
          <div class="chatArea" ref="chatArea">
            <div class="message" v-for="item in chatMsgList" :key="item">
              <div v-html="item"></div>
            </div>
          </div>
        </div>
        <div
          v-if="can(RoomMemberPermission.PermissionSendChatMessage)"
          class="card-footer"
          style="justify-content: center; padding: 0.5rem"
        >
          <input
            type="text"
            @keyup.enter="() => sendChatText(sendChatMsg_, onSendSuccess)"
            v-model="sendChatMsg_"
            placeholder="按 Enter 键即可发送..."
            class="l-input w-full bg-transparent"
            autocomplete="off"
          />
          <button
            class="btn w-24 m-2.5 ml-0"
            @click="() => sendChatText(sendChatMsg_, onSendSuccess)"
          >
            发送
          </button>
        </div>
      </div>
    </el-col>
  </el-row>

  <el-row :gutter="20">
    <!-- 房间信息 -->
    <el-col :lg="6" :md="8" :sm="9" :xs="24" class="mb-5 max-sm:mb-2">
      <RoomInfo v-if="roomInfo" :status="status" :token="token" :roomId="roomID" :info="roomInfo" />
    </el-col>

    <!-- 影片列表 -->
    <el-col
      v-if="can(RoomMemberPermission.PermissionGetMovieList)"
      :lg="12"
      :md="16"
      :sm="15"
      :xs="24"
      class="mb-5 max-sm:mb-2"
    >
      <MovieList @send-msg="sendMsg" :token="token" :roomId="roomID" />
    </el-col>

    <!-- 添加影片 -->
    <el-col
      v-if="can(RoomMemberPermission.PermissionAddMovie)"
      :lg="6"
      :md="14"
      :xs="24"
      class="mb-5 max-sm:mb-2"
    >
      <MoviePush :token="token" :roomId="roomID" />
    </el-col>
  </el-row>
</template>

<style lang="less" scoped>
.chatArea {
  overflow-y: scroll;
  height: 67vh;
  transition: height 0.3s ease;
}

.loading-spinner {
  margin: 0 auto;
  width: 70px;
  text-align: center;

  > div {
    width: 18px;
    height: 18px;
    background-color: #409eff;
    border-radius: 100%;
    display: inline-block;
    animation: bounce 1.4s infinite ease-in-out both;
    margin: 0 3px;
  }

  .bounce1 {
    animation-delay: -0.32s;
  }

  .bounce2 {
    animation-delay: -0.16s;
  }
}

.audio-controls-container {
  transition: all 0.3s ease-in-out;
  transform-origin: top;
  animation: slideDown 0.3s ease-in-out;
}

.video-wall {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 8px;
}

.video-wall .video-tile {
  width: 100%;
  aspect-ratio: 4 / 3;
  object-fit: cover;
  background-color: #000;
  border-radius: 6px;
}

.video-wall .screen-tile {
  grid-column: 1 / -1;
  aspect-ratio: 16 / 9;
  object-fit: contain;
}

.audio-controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 10px;

  .el-select {
    width: 100%;
    animation: fadeIn 0.3s ease-in-out;
  }

  .volume-control {
    animation: fadeIn 0.3s ease-in-out 0.1s;
  }

  .el-button {
    animation: fadeIn 0.3s ease-in-out 0.2s;
  }
}

@keyframes slideDown {
  from {
    transform: scaleY(0);
    opacity: 0;
  }
  to {
    transform: scaleY(1);
    opacity: 1;
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes bounce {
  0%,
  80%,
  100% {
    transform: scale(0);
  }
  40% {
    transform: scale(1);
  }
}
</style>
