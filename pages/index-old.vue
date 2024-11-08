<!-- 98d1d7d1ae4941e98c9edc6e721490b8 -->

<script setup lang="ts">
import AgoraRTC from 'agora-rtc-sdk-ng';
const app_id = '98d1d7d1ae4941e98c9edc6e721490b8';
const token = null;
const rtc_uid = Math.floor(Math.random() * 2032);

let room_id = 'main'
let audio_tracks = {
    local_audio_tracks: null,
    remote_audio_tracks: {

    }
}
let rtc_client;

let initRtc = async () => {
    rtc_client = AgoraRTC.createClient({
        mode: 'rtc',
        codec: 'vp8',
    })
    await rtc_client.join(app_id, room_id, token, rtc_uid)
    audio_tracks.local_audio_tracks = await AgoraRTC.createMicrophoneAudioTrack()
    rtc_client.publish(audio_tracks.local_audio_tracks)

    let user_wrapper = '<div id="' + rtc_uid + '" class="speaker user-' + rtc_uid + '"><p>' + rtc_uid + '</p></div>'
    document.getElementById('members')?.insertAdjacentHTML('beforeend', user_wrapper)
}

const enterRoom = () => {
    initRtc()
}

const leaveRoom = async () => {
    audio_tracks.local_audio_tracks.stop()
    audio_tracks.local_audio_tracks.close()
    rtc_client.unpublish()
    rtc_client.leave()
}
</script>

<template>
    <div>
        <div id="room-header" class="justify-between items-center p-4 px-0">
            <h1 id="room-name"></h1>
            <div id="room-header-controls" class="flex">
                <UButton id="mic-icon" icon="i-ic:baseline-mic" size="sm" color="primary" square variant="solid" />
                <UButton id="mic-off-icon" icon="i-ic:baseline-mic-off" size="sm" color="primary" square
                    variant="solid" />
                <UButton id="leave-icon" icon="i-ic:baseline-phone-disabled" size="sm" color="primary" square
                    variant="solid" @click="leaveRoom" />
            </div>
        </div>

        <form @submit.prevent="enterRoom">
            <UButton type="submit">Submit</UButton>
        </form>

        <div id="members" class="flex flex-wrap"></div>
    </div>
</template>

<style scoped></style>
