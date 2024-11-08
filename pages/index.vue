<script setup lang="ts">
import AgoraRTC from 'agora-rtc-sdk-ng';

// Agora App ID
const appid = '98d1d7d1ae4941e98c9edc6e721490b8';
const token = null;
const rtcUid = Math.floor(Math.random() * 2032);
const roomId = 'main';

// Client and tracks
const rtcClient = AgoraRTC.createClient({ mode: 'rtc', codec: 'vp8' });
const localAudioTrack = ref(null);
const remoteAudioTracks = ref({});
const members = ref([]);
const micMuted = ref(false);
const isInRoom = ref(false);

// Initialize RTC
const initRtc = async () => {
  // Join the room
  await rtcClient.join(appid, roomId, token, rtcUid);

  // Create and publish local audio track
  localAudioTrack.value = await AgoraRTC.createMicrophoneAudioTrack();
  await rtcClient.publish(localAudioTrack.value);

  // Add local user to the members list
  members.value.push({ uid: rtcUid, volumeLevel: 0 });

  // Enable volume indicator
  initVolumeIndicator();

  // Event listeners
  rtcClient.on('user-joined', handleUserJoined);
  rtcClient.on('user-published', handleUserPublished);
  rtcClient.on('user-left', handleUserLeft);
};

// Volume Indicator
const initVolumeIndicator = () => {
  AgoraRTC.setParameter('AUDIO_VOLUME_INDICATION_INTERVAL', 200);
  rtcClient.enableAudioVolumeIndicator();

  rtcClient.on('volume-indicator', (volumes) => {
    volumes.forEach((volume) => {
      const member = members.value.find((m) => m.uid === volume.uid);
      if (member) {
        member.volumeLevel = volume.level;
      }
    });
  });
};

// Handle user joining
const handleUserJoined = (user) => {
  members.value.push({ uid: user.uid, volumeLevel: 0 });
};

// Handle user publishing
const handleUserPublished = async (user, mediaType) => {
  await rtcClient.subscribe(user, mediaType);

  if (mediaType === 'audio') {
    remoteAudioTracks.value[user.uid] = user.audioTrack;
    user.audioTrack.play();
  }
};

// Handle user leaving
const handleUserLeft = (user) => {
  members.value = members.value.filter((member) => member.uid !== user.uid);
  delete remoteAudioTracks.value[user.uid];
};

// Enter room
const enterRoom = async () => {
  await initRtc();
  isInRoom.value = true;
};

// Leave room
const leaveRoom = async () => {
  if (localAudioTrack.value) {
    localAudioTrack.value.stop();
    localAudioTrack.value.close();
  }
  await rtcClient.unpublish();
  await rtcClient.leave();

  isInRoom.value = false;
  members.value = [];
};

// Toggle microphone
const toggleMic = () => {
  if (localAudioTrack.value) {
    micMuted.value = !micMuted.value;
    localAudioTrack.value.setEnabled(!micMuted.value);
  }
};

// Cleanup on unmount
onBeforeUnmount(() => {
  rtcClient.off('user-joined', handleUserJoined);
  rtcClient.off('user-published', handleUserPublished);
  rtcClient.off('user-left', handleUserLeft);
});
</script>

<template>
  <div id="container" class="flex flex-col items-center justify-center min-h-screen p-4">
    <!-- Room Header (Visible when in room) -->
    <div v-show="isInRoom" id="room-header" class="flex w-full max-w-md justify-between items-center mb-4">
      <h1 id="room-name" class="text-lg font-semibold">{{ roomId }}</h1>

      <div id="room-header-controls" class="flex">
        <!-- Mic Toggle Icon -->
        <UButton @click="toggleMic" class="text-gray-700 hover:text-red-500" variant="link">
          <Icon :name="micMuted ? 'i-ph-microphone-slash' : 'i-ph-microphone'" size="24" />
        </UButton>
        <!-- Leave Room Icon -->
        <UButton @click="leaveRoom" class="text-gray-700 hover:text-red-500" variant="link">
          <Icon name="i-ph-sign-out-bold" size="24" />
        </UButton>
      </div>
    </div>

    <!-- Form to Enter the Room (Visible when not in room) -->
    <form v-show="!isInRoom" id="form" @submit.prevent="enterRoom" class="flex flex-col items-center">
      <UButton type="submit">Enter Room</UButton>
    </form>

    <!-- Members List -->
    <div id="members" class="flex flex-col w-full max-w-md gap-2 mt-4">
      <div v-for="member in members" :key="member.uid"
        :class="['speaker', `user-rtc-${member.uid}`, 'border-2 rounded-lg p-2 text-center shadow-sm', member.volumeLevel >= 50 ? 'border-primary' : '']">
        <p class="text-gray-700 font-medium">{{ member.uid }}</p>
      </div>
    </div>
  </div>
</template>