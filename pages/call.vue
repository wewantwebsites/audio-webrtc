<template>
  <div>
    <h1 class="w-full lg:text-center">WebRTC Call</h1>
    <section class="mb-4 flex justify-between lg:justify-around">
      <h2>Start Your Webcam</h2>
      <div>
        <UButton class="mr-4" @click="startWebcam" :disabled="isWebcamOn">Start Webcam</UButton>
        <UButton class="mr-4" variant="outline" @click="endWebcam" :disabled="!isWebcamOn">End Webcam</UButton>
        <UButton color="red" variant="outline" @click="hangup" id="hangUpButton" :disabled="!isCallActive">Hang Up
        </UButton>
      </div>
    </section>

    <section id="videos" class="flex justify-between mb-4 lg:justify-around">
      <div>
        <h3>You</h3>
        <video id="localVideo" :srcObject="localStream" autoplay playsinline class="bg-yellow-500"></video>
      </div>
      <UDivider orientation="vertical" class="hidden lg:block" />
      <div>
        <h3>Them</h3>
        <video id="remoteVideo" :srcObject="remoteStream" autoplay playsinline class=""></video>
      </div>
    </section>
    <UDivider class="mb-4" />
    <section class="mb-4 flex justify-between lg:justify-around">
      <div class="mb-4">
        <h2>Create New Call</h2>
        <UButton @click="createCall" id="callButton" :disabled="isCallActive">Create Call</UButton>
      </div>
      <div>
        <h2>Join Existing Call</h2>
        <div class="flex">

          <UInput class="mr-2" v-model="callInput" id="callInput" placeholder="Paste Call ID"></UInput>

          <UButton @click="answerCall" id="answerButton" :disabled="isCallActive">Answer</UButton>
        </div>
      </div>
    </section>

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
  peerConnection.value.close();
  isCallActive.value = false;
  endWebcam();
}

function endWebcam() {
  localStream.value.getTracks().forEach((track) => track.stop());
  remoteStream.value.getTracks().forEach((track) => track.stop());
  isWebcamOn.value = false;
}
async function startWebcam(e) {
  isWebcamOn.value = true;
  localStream.value = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  });
  remoteStream.value = new MediaStream();

  // push tracks
  localStream.value.getTracks().forEach((track) => {
    peerConnection.value.addTrack(track, localStream);
  });

  // pull tracks
  peerConnection.ontrack = (event) => {
    event.streams[0].getTracks().forEach((track) => {
      remoteStream.value.addTrack(track);
    });
  };

  isWebcamOn.value = true
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
  channel = socket.channel(`call:${callId}`, { type: 'callee' });
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
