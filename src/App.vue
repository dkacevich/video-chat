<template>
    <div class="container">
        <div class="videos">
            <video class="video_offer" :src-object.prop.camel="localStream" autoplay playsinline muted />
            <video class="video_answer" :src-object.prop.camel="remoteStream" autoplay playsinline />
        </div>
        <div class="wrapper">
            <div class="buttons">
                <button @click="initWebCam">Open Webcam</button>
                <button @click="makeCall" :disabled="callDisable">Make call</button>
                <button @click="answerCall" :disabled="answerDisable">Answer call</button>
            </div>
            <input v-model="callInputValue" placeholder="Your call id will generated here">
        </div>
    </div>
</template>




<script setup>
import { onMounted, ref } from 'vue';
import freeice from 'freeice'
import { initializeApp } from "firebase/app";
import { updateDoc, getDoc, onSnapshot, addDoc, getFirestore, collection, doc, setDoc } from "firebase/firestore";

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
    padding: 10px;
}

.videos {
    position: relative;
}


.wrapper {
    width: 100%;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    position: absolute;
    bottom: 0;
    left: 0;
    z-index: 6;
}

.buttons {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
}

button {
    padding: 8px 14px;
    width: 140px;
    background: #021cb0;
    border: none;
    color: #fff;
    font-weight: 700;
    border-radius: 10px;
}

input {
    padding: 10px 20px;
    border-radius: 10px;
    font-size: 18px;
    outline: none;
}

.video_offer {
    max-width: 250px;
    border: 3px solid #ffffff;
    border-radius: 10px;
    overflow: hidden;
    position: absolute;
    top: 0;
    left: 0;
    z-index: 4;
}

.video_answer {
    position: fixed;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    background: #000;
}
</style>