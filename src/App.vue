<template>
  <div id="app">

    <div :class="{'first-search': !results}">
        <form class="form-horizontal" v-on:submit.prevent="searchGIFs()">
            <div class="form-group">
                <h3>Search gifs by key words</h3>
                <input type="text" class="form-control" placeholder="Key words" v-model="query">
            </div>
            <button type="submit" class="btn btn-primary" id="searchBtn" :disabled="query.trim() == '' ? true : false">
                Search
            </button>
        </form>
        <div v-show="results" class="results-box">
            <img :src="gif.images.fixed_height_small.webp" class="image-card" :key="gif.images.fixed_height_small.url" v-for="gif in results" />
        </div>
    </div>

		<form>
			<input type="text" id="query" placeholder="Type the name of a track (e.g. Beyonce - Baby Boy)">
      <input type="submit" value="Search track &amp; Calculate tempo">
		</form>

		<div id="result">
			<div id="text"></div>
			<div>
				<svg width="100%" height="40" id="svg"></svg>
				<button id="play">Play track</button>
			</div>

			<audio id="audio"></audio>
      </div>
      <div  id="app-viewer" class="gifbox" v-show="results">
        <img :src="current_gif" v-show="current_gif" class="center-block">
        <h3 v-show="results.length === 0" class="no-results">No results were found</h3>
        <h3 v-show="results.length > 0 && !current_gif">Select a gif</h3>
    </div>   
  </div>
</template>

<script>

  
const apiEndPoint = 'https://api.giphy.com/v1/gifs/search';
const publicKey = 'dc6zaTOxFJmzC';

import * as Promise from 'bluebird'
import SpotifyWebApi from './spotify-web-api'
import axios from 'axios'

function getPeaks(data) {

  var partSize = 22050,
      parts = data[0].length / partSize,
      peaks = [];

  for (var i = 0; i < parts; i++) {
    var max = 0;
    for (var j = i * partSize; j < (i + 1) * partSize; j++) {
      var volume = Math.max(Math.abs(data[0][j]), Math.abs(data[1][j]));
      if (!max || (volume > max.volume)) {
        max = {
          position: j,
          volume: volume
        };
      }
    }
    peaks.push(max);
  }

  peaks.sort(function(a, b) {
    return b.volume - a.volume;
  });

  peaks = peaks.splice(0, peaks.length * 0.5);

  peaks.sort(function(a, b) {
    return a.position - b.position;
  });

  return peaks;
}

function getIntervals(peaks) {

  var groups = [];

  peaks.forEach(function(peak, index) {
    for (var i = 1; (index + i) < peaks.length && i < 10; i++) {
      var group = {
        tempo: (60 * 44100) / (peaks[index + i].position - peak.position),
        count: 1
      };

      while (group.tempo < 90) {
        group.tempo *= 2;
      }

      while (group.tempo > 180) {
        group.tempo /= 2;
      }

      group.tempo = Math.round(group.tempo);

      if (!(groups.some(function(interval) {
        return (interval.tempo === group.tempo ? interval.count++ : 0);
      }))) {
        groups.push(group);
      }
    }
  });
  return groups;
}

function prefetchImg(url) {
  return new Promise((resolve, reject) => {
    let img = new Image()
    img.onload = () => resolve({ url, status: "ok" })
    img.onerror = () => reject({ url, status: "error" })
    img.src = url
  })
}

export default {
  name: 'app',

  data: function() {
    return {
        query: '',
        results: false,
        current_gif: false,
        peaks: [],
        promises: [],
        lastPeakIndex: 0
      }
    },
    methods: { 
        searchGIFs: function () {
            var self = this;
            axios.get(apiEndPoint, {
                params: {
                    api_key: publicKey,
                    q: self.query.split(' ').join('+'),
                    limit: 50
                }
            })
            .then(function (response) {
                self.results = response.data.data;
                self.current_gif = false;
                if (self.results) {
                  prefetchImg(self.results[0].images.original.webp)
                  prefetchImg(self.results[1].images.original.webp)
                  prefetchImg(self.results[2].images.original.webp)
                  self.promises = Promise.mapSeries(self.results, (item) => { return prefetchImg(item.images.original.webp)})
                }
            })
            .catch(function (error) {
                // eslint-disable-next-line
                console.log(error);
            });
        },
        viewGIF: function (gif) {
            this.current_gif = gif.images.original.webp;
        }
    },

  mounted(){
    var self = this;
    var spotifyApi = new SpotifyWebApi();
    spotifyApi.getToken().then(function(response) {
      spotifyApi.setAccessToken(response.token);
    });

    var queryInput = document.querySelector('#query'),
        result = document.querySelector('#result'),
        text = document.querySelector('#text'),
        audioTag = document.querySelector('#audio'),
        playButton = document.querySelector('#play');

    function updateProgressState() {
      if (audioTag.paused) {
        return;
      }
      var progressIndicator = document.querySelector('#progress');
      if (progressIndicator && audioTag.duration) {
        var position = audioTag.currentTime * 100 / audioTag.duration
        progressIndicator.setAttribute('x', position + '%');        
        if (position+1 > self.peaks[self.lastPeakIndex]) {
          self.lastPeakIndex+=1;

          var gifIndex = self.lastPeakIndex % self.results.length
          self.current_gif = self.results[gifIndex].images.original.webp;
          // eslint-disable-next-line
          console.log('boom')
        }
        if (position > 99) {
          self.lastPeakIndex = 0;
        }

      }
      

      requestAnimationFrame(updateProgressState);
    }

    audioTag.addEventListener('play', updateProgressState);
    audioTag.addEventListener('playing', updateProgressState);

    function updatePlayLabel() {
      playButton.innerHTML = audioTag.paused ? 'Play track' : 'Pause track';

    }

    audioTag.addEventListener('play', updatePlayLabel);
    audioTag.addEventListener('playing', updatePlayLabel);
    audioTag.addEventListener('pause', updatePlayLabel);
    audioTag.addEventListener('ended', updatePlayLabel);

    playButton.addEventListener('click', function() {
      if (audioTag.paused) {
        audioTag.play();
      } else {
        audioTag.pause();
      }
    });

    result.style.display = 'none';


    document.querySelectorAll('form')[1].addEventListener('submit', function(formEvent) {
      formEvent.preventDefault();
      result.style.display = 'none';
      spotifyApi.searchTracks(
        queryInput.value.trim(), {limit: 1})
        .then(function(results) {
          var track = results.tracks.items[0];
          var previewUrl = track.preview_url;
          audioTag.src = track.preview_url;

          var request = new XMLHttpRequest();
          request.open('GET', previewUrl, true);
          request.responseType = 'arraybuffer';
          request.onload = function() {

            // Create offline context
            var OfflineContext = window.OfflineAudioContext || window.webkitOfflineAudioContext;
            var offlineContext = new OfflineContext(2, 30 * 44100, 44100);

            offlineContext.decodeAudioData(request.response, function(buffer) {

              // Create buffer source
              var source = offlineContext.createBufferSource();
              source.buffer = buffer;

              // Beats, or kicks, generally occur around the 100 to 150 hz range.
              // Below this is often the bassline.  So let's focus just on that.

              // First a lowpass to remove most of the song.

              var lowpass = offlineContext.createBiquadFilter();
              lowpass.type = "lowpass";
              lowpass.frequency.value = 150;
              lowpass.Q.value = 1;

              // Run the output of the source through the low pass.

              source.connect(lowpass);

              // Now a highpass to remove the bassline.

              var highpass = offlineContext.createBiquadFilter();
              highpass.type = "highpass";
              highpass.frequency.value = 100;
              highpass.Q.value = 1;

              // Run the output of the lowpass through the highpass.

              lowpass.connect(highpass);

              // Run the output of the highpass through our offline context.

              highpass.connect(offlineContext.destination);

              // Start the source, and render the output into the offline conext.

              source.start(0);
              offlineContext.startRendering();
            });

            offlineContext.oncomplete = function(e) {
              var buffer = e.renderedBuffer;
              var peaks = getPeaks([buffer.getChannelData(0), buffer.getChannelData(1)]);
              var groups = getIntervals(peaks);

              var svg = document.querySelector('#svg');
              svg.innerHTML = '';
              var svgNS = 'http://www.w3.org/2000/svg';
              var rect;
              self.peaks = peaks.map(e => (100 * e.position / buffer.length ) )
              peaks.forEach(function(peak) {
                rect = document.createElementNS(svgNS, 'rect');
                rect.setAttributeNS(null, 'x', (100 * peak.position / buffer.length) + '%');
                rect.setAttributeNS(null, 'y', 0);
                rect.setAttributeNS(null, 'width', 1);
                rect.setAttributeNS(null, 'height', '100%');
                svg.appendChild(rect);
              });

              rect = document.createElementNS(svgNS, 'rect');
              rect.setAttributeNS(null, 'id', 'progress');
              rect.setAttributeNS(null, 'y', 0);
              rect.setAttributeNS(null, 'width', 1);
              rect.setAttributeNS(null, 'height', '100%');
              svg.appendChild(rect);

              svg.innerHTML = svg.innerHTML + ''; // force repaint in some browsers

              var top = groups.sort(function(intA, intB) {
                return intB.count - intA.count;
              }).splice(0, 5);

              text.innerHTML = '<div id="guess">Guess for track <strong>' + track.name + '</strong> by ' +
                '<strong>' + track.artists[0].name + '</strong> is <strong>' + Math.round(top[0].tempo) + ' BPM</strong>' +
                ' with ' + top[0].count + ' samples.</div>';

              text.innerHTML += '<div class="small">Other options are ' +
                top.slice(1).map(function(group) {
                  return group.tempo + ' BPM (' + group.count + ')';
                }).join(', ') +
                '</div>';

              var printENBPM = function(tempo) {
                text.innerHTML += '<div class="small">The tempo according to Spotify is ' +
                      tempo + ' BPM</div>';
              };
              spotifyApi.getAudioFeaturesForTrack(track.id)
                .then(function(audioFeatures) {
                  printENBPM(audioFeatures.tempo);
                });

              result.style.display = 'block';
            };
          };
          request.send();
        });
    });

  }
}
</script>

<style>
* {
	padding: 0;
	margin: 0;
	box-sizing: border-box;
	font-family: Helvetica, Arial;
}

body {
	font-size: 14px;
	line-height: 1.4em;
	padding: 2em;
	max-width: 50em;
	margin: 0 auto;
}

@media (min-width: 600px) {
	body {
		font-size: 16px;
	}
}

a {
	color: #2b7bb9;
	text-decoration: none;
}
h1 {
	line-height: 1.2em;
}

h1, h2, p, form, ul, ol, svg {
	margin-bottom: 1em;
}

ul, ol {
	margin-left: 2em;
}

input[type=text] {
	padding: 0.5em;
	width: 100%;
	border: 1px solid #ddd;
	font-size:1.1em;
	display:block;
	text-align: center;
}
input[type=submit] {
	padding: 0.5em;
	border: 1px solid #ccc;
	background-color: #ddd;
	cursor: pointer;
	font-size:1em;
	display:block;
	width: 100%
}

#guess, #text {
	margin-bottom: 1em;
}

#play {
	padding: 0.5em;
	border: 1px solid #ccc;
	background-color: #ddd;
	cursor: pointer;
	font-size:1em;
	display:block;
	width: 100%
}

#result {
	padding: 1em;
	font-size: 1.3em;
	margin-bottom: 1em;
}

svg {
	border: 1px solid #eee;
}

.gifbox {
  position: absolute;
  top: 10px;
  left: 10px;
  width: 500px;
}

.gifbox img {
  width: 100%;
}

svg #progress {
	fill: red;
}

.small {
	font-size: 0.8em;
}

form {
	padding: 0.5em;
}

/* FOOTER BOXES */
footer {
	border-top: 1px solid rgba(0,0,0,0.15);
	padding: 1em;
}

footer h3 {
	color: rgba(0,0,0,0.3);
	text-transform: uppercase;
	font-size: 80%;
	margin-bottom: 0.5em;
}

/* MEDIA OBJECT */
.media, .bd {overflow:hidden; _overflow:visible; zoom:1;}
.media .img {float:left;}

/* ABOUT BOX */
.about .avatar-image {
	border-radius: 100%;
	height: 80px;
	width: 80px;
	margin-right: 10px;
}

.about .name {
	font-weight: bold;
	font-size: 110%;
	margin-top: 5px;
	margin-bottom: 2px;
}

.about > .media {
	margin-top: 10px;
}

.twitter {
	margin-bottom: 3px;
}
.twitter-logo {
	margin-top: 2px;
}
</style>
