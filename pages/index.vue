<script setup>
import AgoraRTC from 'agora-rtc-sdk-ng';

const appId = '98d1d7d1ae4941e98c9edc6e721490b8';
const token = null;
const rtcUid = Math.floor(Math.random() * 2032);
const roomId = 'main';

const rtcClient = AgoraRTC.createClient({ mode: 'rtc', codec: 'vp8' });

const micMuted = ref(true);
const audioTracks = ref({
  localAudioTrack: null,
  remoteAudioTracks: {},
});

const members = ref([]);
const isInRoom = ref(false);

const initRtc = async () => {
  rtcClient.on('user-joined', handleUserJoined);
  rtcClient.on('user-published', handleUserPublished);
  rtcClient.on('user-left', handleUserLeft);

  await rtcClient.join(appId, roomId, token, rtcUid);
  audioTracks.value.localAudioTrack = await AgoraRTC.createMicrophoneAudioTrack();
  audioTracks.value.localAudioTrack.setMuted(micMuted.value);
  await rtcClient.publish(audioTracks.value.localAudioTrack);

  members.value.push({ uid: rtcUid, isLocal: true });
  initVolumeIndicator();
};

const initVolumeIndicator = () => {
  AgoraRTC.setParameter('AUDIO_VOLUME_INDICATION_INTERVAL', 200);
  rtcClient.enableAudioVolumeIndicator();

  rtcClient.on('volume-indicator', (volumes) => {
    volumes.forEach((volume) => {
      const member = members.value.find((m) => m.uid === volume.uid);
      if (member) {
        member.isTalking = volume.level >= 50;
      }
    });
  });
};

const handleUserJoined = (user) => {
  members.value.push({ uid: user.uid, isLocal: false });
};

const handleUserPublished = async (user, mediaType) => {
  await rtcClient.subscribe(user, mediaType);
  if (mediaType === 'audio') {
    audioTracks.value.remoteAudioTracks[user.uid] = user.audioTrack;
    user.audioTrack.play();
  }
};

const handleUserLeft = (user) => {
  members.value = members.value.filter((member) => member.uid !== user.uid);
  delete audioTracks.value.remoteAudioTracks[user.uid];
};

const toggleMic = async () => {
  micMuted.value = !micMuted.value;
  await audioTracks.value.localAudioTrack.setMuted(micMuted.value);
};

const enterRoom = async () => {
  await initRtc();
  isInRoom.value = true;
};

const leaveRoom = async () => {
  audioTracks.value.localAudioTrack.stop();
  audioTracks.value.localAudioTrack.close();
  await rtcClient.unpublish();
  await rtcClient.leave();

  members.value = [];
  isInRoom.value = false;
};
</script>

<template>
  <div class="flex flex-col items-center justify-center min-h-screen bg-gray-800 text-white">
    <div v-if="!isInRoom" class="flex flex-col items-center">
      <h1 class="text-2xl font-bold mb-4">Join Agora Voice Chat</h1>
      <UButton @click="enterRoom">Enter Room</UButton>
    </div>

    <div v-else class="w-full max-w-lg">
      <div class="flex justify-between items-center p-4 bg-gray-700 rounded-t-lg">
        <h2 class="text-lg font-semibold">Room: {{ roomId }}</h2>
        <div class="flex items-center">
          <UButton @click="toggleMic" variant="link" :icon="micMuted ? 'i-ph-microphone-slash' : 'i-ph-microphone'" />
          <UButton @click="leaveRoom" variant="link" icon="i-ph-sign-out-bold" color="red" />
        </div>
      </div>

      <div class="p-4 bg-gray-600 rounded-b-lg space-y-4">
        <div v-for="member in members" :key="member.uid" :class="[
          'flex items-center justify-between p-2 rounded-lg',
          member.isLocal ? 'bg-blue-600' : 'bg-gray-600',
          member.isTalking ? 'border-2 border-green-500' : 'border-2 border-transparent'
        ]">
          <span>User: {{ member.uid }}</span>
          <span v-if="member.isLocal">(You)</span>
        </div>
      </div>
    </div>
  </div>
</template>