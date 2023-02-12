<template>
    <div class="container">
        <div class="videos">
            <video class="video" :src-object.prop.camel="localStream" autoplay playsinline muted />
            <video class="video" :src-object.prop.camel="remoteStream" autoplay playsinline />
        </div>
        <div class="buttons">
            <button @click="initWebCam">Open Webcam</button>
            <button @click="makeCall" :disabled="callDisable">Make call</button>
            <input v-model="callInputValue" placeholder="Your call id will generated here">
            <button @click="answerCall" :disabled="answerDisable">Answer call</button>
        </div>
    </div>
</template>




<script setup>
import { onMounted, ref } from 'vue';
import freeice from 'freeice'
import { initializeApp } from "firebase/app";
import { updateDoc, getDoc, getDocs, onSnapshot, addDoc, getFirestore, collection, doc, setDoc } from "firebase/firestore";

const firebaseConfig = {
    apiKey: "AIzaSyBCDi_qqSiTXNvnNtQIErHpUB9EGoahQ5Q",
    authDomain: "video-1-dc27b.firebaseapp.com",
    projectId: "video-1-dc27b",
    storageBucket: "video-1-dc27b.appspot.com",
    messagingSenderId: "20894049667",
    appId: "1:20894049667:web:bd94143d6c39bbddc302b6"
};


const localStream = ref()
const remoteStream = ref()
const db = ref()
const callInputValue = ref()
const callDisable = ref(true)
const answerDisable = ref(true)
const pc = ref(new RTCPeerConnection(freeice()))



const initWebCam = async () => {
    localStream.value = await navigator.mediaDevices.getUserMedia({ video: true, audio: true })
    remoteStream.value = new MediaStream()

    localStream.value.getTracks().forEach(track => {
        pc.value.addTrack(track, localStream.value)
    })

    pc.value.ontrack = event => {
        event.streams[0].getTracks().forEach(track => {
            console.log(track);
            remoteStream.value.addTrack(track);
        });
    };

    callDisable.value = false
    answerDisable.value = false
}



onMounted(async () => {
    db.value = getFirestore(initializeApp(firebaseConfig))
})


const makeCall = async () => {
    const callDoc = doc(collection(db.value, "calls"))

    const offerCandidates = collection(callDoc, 'offerCandidates')
    const answerCandidates = collection(callDoc, 'answerCandidates')

    callInputValue.value = callDoc.id


    pc.value.onicecandidate = e => {
        if (e.candidate) addDoc(offerCandidates, e.candidate.toJSON())
    }


    const offerDescription = await pc.value.createOffer()
    await pc.value.setLocalDescription(offerDescription)


    const offer = {
        sdp: offerDescription.sdp,
        type: offerDescription.type
    }


    await setDoc(callDoc, { offer })

    callDisable.value = true
    answerDisable.value = true


    onSnapshot(callDoc, (snapshot) => {
        const data = snapshot.data()
        if (!pc.value.currentRemoteDescription && data?.answer) {
            const answerDescription = new RTCSessionDescription(data.answer)
            pc.value.setRemoteDescription(answerDescription)
        }
    })

    onSnapshot(answerCandidates, (snapshot) => {
        snapshot.docChanges().forEach(change => {
            if (change.type === 'added') {
                const candidate = new RTCIceCandidate(change.doc.data())
                pc.value.addIceCandidate(candidate)
            }
        })
    })
}

const answerCall = async () => {
    const callDoc = doc(collection(db.value, 'calls'), callInputValue.value)
    const answerCandidates = collection(callDoc, 'answerCandidates')
    const offerCandidates = collection(callDoc, 'offerCandidates')

    pc.value.onicecandidate = e => {
        if (e.candidate) addDoc(answerCandidates, e.candidate.toJSON())
    }

    const callData = (await getDoc(callDoc)).data()

    await pc.value.setRemoteDescription(new RTCSessionDescription(callData.offer))

    const answerDescription = await pc.value.createAnswer()
    await pc.value.setLocalDescription(answerDescription)

    const answer = {
        sdp: answerDescription.sdp,
        type: answerDescription.type
    }

    await updateDoc(callDoc, { answer })


    onSnapshot(offerCandidates, (snapshot) => {
        snapshot.docChanges().forEach(change => {
            console.log(change);
            if (change.type === 'added') {
                let data = change.doc.data()
                pc.value.addIceCandidate(new RTCIceCandidate(data))
            }
        })
    })
}





</script>

<style lang="scss">
* {
    padding: 0;
    margin: 0;
    box-sizing: border-box;
    font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
}

body {
    background: #011741;
}



.container {
    max-width: 1200px;
    margin: 0px auto;
    padding: 0 15px;
}

.videos {
    margin-top: 100px;
    display: grid;
    gap: 50px;
    justify-content: center;

    @media (min-width: 700px) {
        grid-template-columns: repeat(2, 1fr);
    }
}


.buttons {
    margin-top: 20px;
    gap: 10px;
    display: flex;
    flex-direction: column;
    align-items: center;
}


.video {
    max-width: 100%;
    border: 3px solid #ffffff;
    border-radius: 10px;
    overflow: hidden;
}
</style>