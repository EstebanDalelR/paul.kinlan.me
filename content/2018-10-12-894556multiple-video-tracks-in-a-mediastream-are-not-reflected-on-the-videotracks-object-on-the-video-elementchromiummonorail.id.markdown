---
slug: crbug-894556-multiple-video-tracks-in-a-mediastream-are-not-reflected-on-the-videotracks-object-on-the-video-element
date: 2018-10-12T06:35:22.116Z
title: '894556 - Multiple video tracks in a MediaStream are not reflected on the videoTracks object on the video element'
link: https://bugs.chromium.org/p/chromium/issues/detail?id=894556&can=1&q=reporter%3Ame&colspec=ID%20Pri%20M%20Stars%20ReleaseBlock%20Component%20Status%20Owner%20Summary%20OS%20Modified&desc=3
tags: [links]
---
Masalah pertama yang saya temukan mencoba [membangun editor video di web](https://paul.kinlan.me/building-a-video-editor-on-the-web-with-the-web/).

Saya memiliki beberapa aliran video (desktop dan web cam) dan saya ingin dapat beralih antara aliran video pada satu elemen video sehingga saya dapat dengan cepat beralih antara web cam dan desktop dan tidak melanggar `MediaRecorder`.

Sepertinya Anda harus dapat melakukannya melalui toggling properti `selected` pada objek` videoTracks` pada ` <video> `elemen, tetapi Anda tidak bisa, array trek hanya berisi 1 elemen (trek video pertama di MediaStream).

> What steps will reproduce the problem?
> (1) Get two MediaStreams with video tracks
> (2) Add them to a new MediaStream and attach as srcObject on a videoElement
> (3) Check the videoElement.videoTracks object and see there is only one track
> 
> Demo at https://multiple-tracks-bug.glitch.me/
> 
> What is the expected result?
> I would expect videoElement.videoTracks to have two elements.
> 
> What happens instead?
> It only has the first videoTrack that was added to the MediaStream.


[Baca pos lengkap](https://bugs.chromium.org/p/chromium/issues/detail?id=894556&can=1&q=reporter%3Ame&colspec=ID%20Pri%20M%20Stars%20ReleaseBlock%20Component%20Status%20Owner%20Summary%20OS%20Modified&desc=3).

Repro case.


```javascript
window.onload = () => {
  if('getDisplayMedia' in navigator) warning.style.display = 'none';

  let blobs;
  let blob;
  let rec;
  let stream;
  let webcamStream;
  let desktopStream;

  captureBtn.onclick = async () => {

       
    desktopStream = await navigator.getDisplayMedia({video:true});
    webcamStream = await navigator.mediaDevices.getUserMedia({video: { height: 1080, width: 1920 }, audio: true});
    
    // Always 
    let tracks = [...desktopStream.getTracks(), ... webcamStream.getTracks()]
    console.log('Tracks to add to stream', tracks);
    stream = new MediaStream(tracks);
    
    console.log('Tracks on stream', stream.getTracks());
    
    videoElement.srcObject = stream;
    
    console.log('Tracks on video element that has stream', videoElement.videoTracks)
    
    // I would expect the length to be 2 and not 1
  };

};
```