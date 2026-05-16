<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {}

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })

          return
        }

        data = await response.json()
        this.playerResponse = data
      } catch (error) {
        this.handleExpiredToken()

        data = this.getEmptyPlayer()
        this.playerData = data

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover by replicating the exact Python K-Means workflow.
     */
    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        return
      }

      const imageUrl = this.player.trackAlbum.image;
      const img = new Image();
      img.crossOrigin = "anonymous"; // Enables secure pixel extractions on Spotify's open CDN
      img.src = imageUrl;

      img.onload = () => {
        // Step 1: Downsample image to 100x100 matrix matching Python's image.resize((100, 100))
        const canvas = document.createElement('canvas');
        canvas.width = 100;
        canvas.height = 100;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, 100, 100);

        // Step 2: Extract raw RGB pixel linear matrix
        const imgData = ctx.getImageData(0, 0, 100, 100).data;
        const pixels = [];
        for (let i = 0; i < imgData.length; i += 4) {
          pixels.push([imgData[i], imgData[i+1], imgData[i+2]]);
        }

        // Step 3: Pure Deterministic K-Means Clustering (k=8)
        const k = 8;
        let centroids = [];
        const step = Math.floor(pixels.length / k);
        for (let i = 0; i < k; i++) {
          centroids.push([...pixels[i * step]]);
        }

        let counts = new Array(k).fill(0);
        let labels = new Array(pixels.length);

        // Run iterations to isolate the central clusters
        for (let iter = 0; iter < 10; iter++) {
          counts.fill(0);
          for (let i = 0; i < pixels.length; i++) {
            const p = pixels[i];
            let minDist = Infinity;
            let closestIdx = 0;
            for (let j = 0; j < k; j++) {
              const c = centroids[j];
              const dist = Math.pow(p[0] - c[0], 2) + Math.pow(p[1] - c[1], 2) + Math.pow(p[2] - c[2], 2);
              if (dist < minDist) {
                minDist = dist;
                closestIdx = j;
              }
            }
            labels[i] = closestIdx;
            counts[closestIdx]++;
          }

          let newCentroids = Array.from({ length: k }, () => [0, 0, 0]);
          for (let i = 0; i < pixels.length; i++) {
            const label = labels[i];
            newCentroids[label][0] += pixels[i][0];
            newCentroids[label][1] += pixels[i][1];
            newCentroids[label][2] += pixels[i][2];
          }

          for (let j = 0; j < k; j++) {
            if (counts[j] > 0) {
              newCentroids[j][0] = Math.round(newCentroids[j][0] / counts[j]);
              newCentroids[j][1] = Math.round(newCentroids[j][1] / counts[j]);
              newCentroids[j][2] = Math.round(newCentroids[j][2] / counts[j]);
            }
          }
          centroids = newCentroids;
        }

        // Step 4: Map cluster metrics and sort descending by size/population density
        const palette = centroids.map((c, idx) => ({ rgb: c, count: counts[idx] }));
        palette.sort((a, b) => b.count - a.count);

        // Step 5: Exact Python is_monochrome filter match
        const isMonochrome = (r, g, b) => {
          const max_c = Math.max(r, g, b);
          const min_c = Math.min(r, g, b);
          const diff = max_c - min_c;

          if (diff < 30) return true;
          if (max_c < 30 || min_c > 220) return true;

          return false;
        };

        // Step 6: Exact Python make_vivid pipeline match
        const makeVividHex = (r, g, b) => {
          r /= 255; g /= 255; b /= 255;
          const max = Math.max(r, g, b), min = Math.min(r, g, b);
          let h, s, v = max;
          const d = max - min;
          s = max === 0 ? 0 : d / max;

          if (max === min) {
            h = 0;
          } else {
            switch (max) {
              case r: h = (g - b) / d + (g < b ? 6 : 0); break;
              case g: h = (b - r) / d + 2; break;
              case b: h = (r - g) / d + 4; break;
            }
            h /= 6;
          }

          s = Math.max(s, 0.95);
          v = Math.max(v, 0.95);

          let r2, g2, b2;
          const i = Math.floor(h * 6);
          const f = h * 6 - i;
          const p = v * (1 - s);
          const q = v * (1 - f * s);
          const t = v * (1 - (1 - f) * s);
          switch (i % 6) {
            case 0: r2 = v; g2 = t; b2 = p; break;
            case 1: r2 = q; g2 = v; b2 = p; break;
            case 2: r2 = p; g2 = v; b2 = t; break;
            case 3: r2 = p; g2 = q; b2 = v; break;
            case 4: r2 = t; g2 = p; b2 = v; break;
            case 5: r2 = v; g2 = p; b2 = q; break;
          }

          const rgb = [Math.round(r2 * 255), Math.round(g2 * 255), Math.round(b2 * 255)];
          return "#" + rgb.map(x => x.toString(16).padStart(2, '0')).join('');
        };

        let finalHex = null;
        for (const item of palette) {
          const [r, g, b] = item.rgb;
          if (!isMonochrome(r, g, b)) {
            finalHex = makeVividHex(r, g, b);
            break;
          }
        }

        if (!finalHex) {
          finalHex = "#282828"; // Exact (40, 40, 40) Python fallback color
        }

        // Mock object payload to pass color formatting to the application state smoothly
        const mockPalette = {
          Vibrant: {
            getHex: () => finalHex,
            getTitleTextColor: () => "#ffffff",
            hex: finalHex
          }
        };

        this.handleAlbumPalette(mockPalette);
      };

      img.onerror = () => {
        this.colourPalette = { name: 'Vibrant', background: "#282828", text: "#ffffff" };
        this.setAppColours();
      };
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()

        return
      }

      /**
       * Player is active, but user has paused.
       */
      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()

        return
      }

      /**
       * The newly fetched track is the same as our stored
       * one, we don't want to update the DOM yet.
       */
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      /**
       * Store the current active track.
       */
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    /**
     * Handle newly stored colour palette:
     * - Map data to readable format
     * - Get and store random colour combination.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(key => palette[key] !== null)
        .map(key => {
          return {
            name: key, 
            text: palette[key].getTitleTextColor(),
            background: palette[key].getHex()
          }
        });

      this.swatches = albumColours;

      const dominant = albumColours.find(c => c.name === 'Vibrant') || albumColours[0];

      this.colourPalette = dominant;

      this.$nextTick(() => {
        this.setAppColours();
      });
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },
  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)

      this.$nextTick(() => {
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>