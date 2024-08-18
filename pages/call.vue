<template>
  <div>
    <h2>Start Your Webcam</h2>
    <section id="videos">
      <div>
        <h3>You</h3>
        <video id="localVideo" :srcObject="localStream" autoplay playsinline></video>
      </div>
      <div>
        <h3>Them</h3>
        <video id="remoteVideo" :srcObject="remoteStream" autoplay playsinline></video>
      </div>
    </section>
    <UButton @click="startWebcam" id="webcamButton" :disabled="isWebcamOn">Start Webcam</UButton>
    <UDivider />

    <h2>Create New Call</h2>
    <UButton @click="createCall" id="callButton" :disabled="isCallActive">Create Call</UButton>
    <UDivider />

    <!-- <h2>Join Existing Call</h2>
    <UInput v-model="callInput" id="callInput" placeholder="Paste Call ID"></UInput>
    <UButton id="answerButton" :disabled="!isCallActive">Answer</UButton>
    <UDivider /> -->

    <h2>Hang Up</h2>
    <UButton @click="hangup" id="hangUpButton" :disabled="!isCallActive">Hang Up</UButton>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const isWebcamOn = ref(false)
const isCallActive = ref(false);
const callInput = ref('');
const localStream = ref(null);
const remoteStream = ref(null);
const peerConnection = ref(null);

const servers = {
  iceServers: [
    {
      urls: ['stun:stun.l.google.com:19302', 'stun:stun2.1.google.com:19302'],
    },
  ],
  iceCandidatePoolSize: 10,
};

onMounted(() => {
  peerConnection.value = new RTCPeerConnection(servers);
});

function hangup() {
  location.reload();
}

async function startWebcam() {
  localStream.value = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  });
  remoteStream.value = new MediaStream();

  // push tracks
  localStream.value.getTracks().forEach(track => {
    peerConnection.value.addTrack(track, localStream);
  });

  // pull tracks
  peerConnection.ontrack = (event) => {
    event.streams[0].getTracks().forEach((track) => {
      remoteStream.value.addTrack(track);
    });
  };

  isWebcamOn.value = true;
}

async function createCall() {
  const callId = crypto.randomUUID();

  channel = socket.channel(`call:${callId}`, { type: 'caller' });
  await join_channel(channel, callId);

  callInput.value = callId;

  peerConnection.value.onicecandidate = (event) => {
    if (!event?.candidate) return;
    channel.push('ice_candidate', {
      ice_candidate: event.candidate,
      type: 'caller',
    }, 10000);
    console.log('Sent ICE candidate:\n', event.candidate);
  };

  isCallActive.value = true;
}
</script>
