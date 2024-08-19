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

    <h2>Join Existing Call</h2>
    <UInput v-model="callInput" id="callInput" placeholder="Paste Call ID"></UInput>
    <UButton @click="answerCall" id="answerButton" :disabled="isCallActive">Answer</UButton>
    <UDivider />

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
  console.group('Creating call');
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
  const offerDescription = await peerConnection.value.createOffer();
  await peerConnection.value.setLocalDescription(offerDescription);

  const offer = {
    sdp: offerDescription.sdp,
    type: offerDescription.type,
  };

  channel.push('offer', { offer }, 10000);
  console.log('Sent offer:\n', offer);
  console.groupEnd();

  channel.on('answer', ({ answer }) => {
    console.log('received answer:\n', answer);
    const answerDescription = new RTCSessionDescription(answer);
    isCallActive.value = true;
  })
  channel.on('ice_candidate', ({ ice_candidate }) => {
    console.log('Received ICE candidate:\n', ice_candidate);
    const candidate = new RTCIceCandidate(ice_candidate);
    peerConnection.value.addIceCandidate(candidate);

  })

}

async function answerCall() {
  console.group('Answering call');
  const callId = callInput.value;
  let callerCandidates;

  // join the channel topic `call:${callId}` on the signalling server
  // this cahnnel will be used for all signalling messages
  cahnnel = socket.channel(`call:${callId}`, { type: 'callee' });
  callerCandidates = await join_channel(channel, callId);
  console.log(`Got ${callerCandidates.length} ICE candidates from caller on channel join`, callerCandidates);

  const offer = await new Promise((res, rej) => {
    return channel.push("get_offer", {}, 10000)
      .receive("ok", (offer) => resolve(offer))
      .receive("error", (error) => reject(error))
      .receive("timeout", () => reject("Networking issue..."));
  })

  console.log('Got offer from caller', offer);

  if (!offer) {
    cannerl.leave();
    alert("Could not fetch offer, invalid call id");
    location.reload();
  }

  peerConnection.value.onicecandidate = (event) => {
    if (event.candidate) {
      channel.push('ice_candidate', {
        ice_candidate: event.candidate,
        type: 'callee',
      }, 10000);
    }
  };

  const offerDesc = offer;
  await peerConnection.value.setRemoteDescription(new RTCSessionDescription(offerDesc));

  const answerDescription = await peerConnection.value.createAnswer();

  await peerConnection.value.setLocalDescription(answerDescription);

  const answer = {
    sdp: answerDescription.sdp,
    type: answerDescription.type,
  };

  channel.push('answer', { answer }, 10000);
  console.log('Sent answer:\n', answer);
  console.groupEnd();

  callerCandidates.forEach((candidate) => {
    console.log('Adding ICE candidate from caller', candidate);
    const iceCandidate = new RTCIceCandidate(candidate);
    peerConnection.value.addIceCandidate(iceCandidate);
  });

  channel.on('ice_candidate', ({ ice_candidate }) => {
    console.log('Received ICE candidate:\n', ice_candidate);
    const candidate = new RTCIceCandidate(ice_candidate);
    peerConnection.value.addIceCandidate(candidate);
  });

  isCallActive.value = true;
}
</script>
