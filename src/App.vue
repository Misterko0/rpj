<template>
  <div class="container">
    <div class="sidebar" :class="{ 'sidebar--open': isSidebarOpen }">
      <button class="sidebar__toggle" @click="toggleSidebar">
        <span v-if="isSidebarOpen"></span>
        <span v-else></span>
      </button>
      <h1>
        <strong> Ročníkový projekt </strong>
      </h1>
      <h2>
        <button class="zelene" @click="showFavoriteSongs">
          Obľúbené skladby
        </button>
      </h2>
      <ul>
        <li v-for="favorite in favorites" :key="favorite.trackId">
          {{ favorite.trackName }} - {{ favorite.artistName }}
          <button @click="removeFavorite(favorite)">x</button>
        </li>
      </ul>
      <h2>História hľadania</h2>
      <ul>
        <li v-for="search in searchHistoryLimited" :key="search.id">
          {{ search.query }} - {{ search.timestamp }}
        </li>
      </ul>
      <h2>Môj playlist</h2>
      <ul>
        <li v-for="song in playlist" :key="song.trackId">
          {{ song.trackName }} - {{ song.artistName }}
          <button @click="removeFromPlaylist(song)">x</button>
        </li>
      </ul>
      <button @click="downloadPlaylist" class="zelene" :disabled="isPlaylistEmpty">
        Stiahnuť playlist
      </button>
      <h2>Žánre</h2>
      <ul>
         <li v-for="genre in genres" :key="genre"><a @click="filterByGenre(genre)" :class="{'genre-link': true, 'active': isActive(genre)}">
    {{ genre }}
  </a>
</li>
</ul>
    </div>
    <div class="main" :class="{ 'main--shifted': isSidebarOpen }">
      <h1 class="title">Songify</h1>
      <form @submit.prevent="search">
        <label for="query">Hľadanie:</label>
        <input type="text" v-model="query" id="query" :placeholder="placeholder"/>
        <button type="submit" class="search-btn">Vyhľadaj TU</button>
      </form>
      <div class="song-cards" :class="{ 'song-cards--shifted': isSidebarOpen }">
        <div v-for="song in visibleSongs" :key="song.trackId" class="song-card">
          <div class="song-card__img">
            <img :src="song.artworkUrl100" :alt="song.trackName" />
          </div>
          <div class="song-card__info">
            <h2 class="song-card__title">{{ song.trackName }}</h2>
            <h3 class="song-card__artist">{{ song.artistName }}</h3>
            <audio :src="song.previewUrl" controls @play="stopAllOtherSongs(song)"></audio>
            <button @click="addToFavorites(song)" class="add-to-favorites" :class="{ 'add-to-favorites--added': song.addedToFavorites }">
              {{ song.addedToFavorites ? "Pridané" : "Pridať do obľúbených" }}
            </button>
            <button @click="addToPlaylist(song)" class="add-to-playlist" style="margin: 1px">
              Pridať do playlistu
            </button>
          </div>
        </div>
      </div>
      <button v-if="moreSongs" class="showMore" @click="showMoreSongs">
        Načítaj viac
      </button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      query: "",
      songs: [],
      visibleSongs: [],
      favorites: [],
      searchHistory: [],
      genres: ["All","Pop","Rock","Hip hop","Electronic","Classical","Country","Jazz","R&B/Soul","Reggae","Folk"],
      isSidebarOpen: false,
      searchLimit: 3,
      moreSongs: false,
      placeholder: "Vyhľadaj skladby",
      lastSearchedQuery: "",
      currentSong: null,
      playlist: [],
      currentGenre: '',
    };
  },
  computed: {
    searchHistoryLimited() {
      return this.searchHistory.slice(0, this.searchLimit);
    },
    isPlaylistEmpty() {
      return this.playlist.length === 0;
    },
    favoriteSongs() {
      return this.songs.filter((song) => this.favorites.includes(song));
    },
  },
  methods: {
    async search() {
      if (this.query === "") return;

      this.lastSearchedQuery = this.query;
      this.moreSongs = true;

      try {
        const response = await fetch(
          `https://itunes.apple.com/search?term=${this.query}&entity=song&limit=50`
        );
        const data = await response.json();
        this.songs = data.results;
        this.visibleSongs = this.songs.slice(0, 3);
        this.searchHistory.unshift({
          id: Date.now(),
          query: this.query,
          timestamp: new Date().toLocaleString(),
        });
      } catch (error) {
        console.error(error);
      }
    },

    addToPlaylist(song) {
      if (!this.playlist.includes(song)) {
        this.playlist.push(song);
      }
    },

    removeFromPlaylist(song) {
      const index = this.playlist.indexOf(song);
      if (index > -1) {
        this.playlist.splice(index, 1);
      }
    },

    playSong(song) {
      if (this.currentSong) {
        this.currentSong.pause();
        this.currentSong = null;
      }
      this.currentSong = new Audio(song.previewUrl);
      this.currentSong.play();
    },

    stopAllOtherSongs(song) {
      const audios = document.getElementsByTagName("audio");
      for (let i = 0, len = audios.length; i < len; i++) {
        if (audios[i].src !== song.previewUrl) {
          audios[i].pause();
        }
      }
    },

    removeFavorite(favorite) {
      const index = this.favorites.indexOf(favorite);
      this.favorites.splice(index, 1);
    },

    showMoreSongs() {
      const visibleCount = this.visibleSongs.length;
      const remainingSongs = this.songs.slice(visibleCount, visibleCount + 3);

      this.visibleSongs = [...this.visibleSongs, ...remainingSongs];
      this.moreSongs = this.visibleSongs.length < this.songs.length;
    },

    showFavoriteSongs() {
      this.visibleSongs = this.favoriteSongs;
    },

    toggleSidebar() {
      this.isSidebarOpen = !this.isSidebarOpen;
    },

    filterByGenre(genre) {
    this.currentGenre = genre;
    this.visibleSongs = this.songs.filter((song) => {
      return genre === "All" || song.primaryGenreName === genre;
    }).slice(0, 3);
  },

    favoriteSongs() {
      return this.songs
        .map((song) => ({
          ...song,
          addedToFavorites: this.favorites.some(
            (favorite) => favorite.trackId === song.trackId
          ),
        }))
        .filter((song) => song.addedToFavorites);
    },

    addToFavorites(song) {
      if (!song.addedToFavorites) {
        song.addedToFavorites = true;
        this.favorites.push(song);
      } else {
        const index = this.favorites.findIndex(
          (favorite) => favorite.trackId === song.trackId
        );
        if (index !== -1) {
          song.addedToFavorites = false;
          this.favorites.splice(index, 1);
        }
      }
    },

    isActive(genre) {
    return this.currentGenre === genre || (this.currentGenre === "" && genre === "");
  },

    favoriteSongs() {
      return this.songs
        .map((song) => ({
          ...song,
          addedToFavorites: this.favorites.some(
            (favorite) => favorite.trackId === song.trackId
          ),
        }))
        .filter((song) => song.addedToFavorites);
    },

    downloadPlaylist() {
      let playlistString = "";
      for (let i = 0; i < this.playlist.length; i++) {
        const song = this.playlist[i];
        const artistName = song.artistName;
        const trackName = song.trackName;
        const previewUrl = song.previewUrl;
        playlistString += `${
          i + 1
        }. ${trackName} - ${artistName} (${previewUrl})\n`;
      }
      const blob = new Blob([playlistString], { type: "text/plain" });
      const url = URL.createObjectURL(blob);
      const link = document.createElement("a");
      link.download = "playlist.txt";
      link.href = url;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    },
  },
};
</script>

<style>
@import url("https://fonts.googleapis.com/css?family=Montserrat&display=swap");
body {
  font-family: "Montserrat", sans-serif;
  background-image: url("https://source.unsplash.com/1600x900/?music");
  background-size: cover;
  background-position: center;
  animation: slide 20s infinite alternate linear;
}

@keyframes slide {
  from {
    background-position: 0 0;
  }
  to {
    background-position: 100% 0;
  }
}

.main {
  margin-left: 0;
  transition: margin-left 0.3s ease-in-out;

  padding-left: 1px;
  padding-left: 20px;
  padding-right: 20px;
  box-sizing: border-box;
}

.main--shifted {
  margin-left: 300px;
  transition: margin-left 0.3s ease-in-out;
}

.container {
  color: #fff;
}

.title {
  font-size: 48px;
  margin-top: 40px;
  margin-bottom: 40px;
  text-align: center;
  text-transform: uppercase;
}

form {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 40px;
}

label {
  margin-right: 10px;
  font-size: 24px;
}

input {
  font-size: 24px;
  padding: 10px;
  border: none;
  border-radius: 20px;
  width: 400px;
  max-width: 100%;
  outline: none;
}

button[type="submit"] {
  font-size: 24px;
  padding: 10px 20px;
  border: none;
  border-radius: 20px;
  background-color: #1db954;
  color: #fff;
  cursor: pointer;
  outline: none;
}

.zelene {
  font-size: 24px;
  padding: 10px 10px;
  border: none;
  border-radius: 20px;
  background-color: #1db954;
  color: #fff;
  cursor: pointer;
}

.song-cards {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
}

.song-card {
  width: 100%;
  margin-bottom: 25px;
  background-color: #282828;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

.song-card__img {
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #000;
}

.song-card__img img {
  width: 100%;
  max-width: 300px;
}

.song-card__info {
  padding: 20px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  flex-grow: 1;
}

.song-card__title {
  font-size: 24px;
  margin-top: 0;
  margin-bottom: 10px;
}

.song-card__artist {
  font-size: 18px;
  margin: 0;
  margin-bottom: 20px;
}

.song-cards--shifted .song-card {
  max-width: calc(33.33% - 20px);
}

@media screen and (min-width: 768px) {
  .song-card {
    flex-direction: row;
  }
}

.song-card__img {
  flex: 1 0 0;
}

@media screen and (min-width: 768px) {
  .song-card__img {
    flex: 0 0 200px;
  }
}

.song-card__info {
  flex: 2 0 0;
}

@media screen and (min-width: 768px) {
  .song-card__info {
    flex: 3 0 0;
  }
}

@media screen and (min-width: 768px) {
  .song-card {
    width: calc(33.33% - 50px);
    margin-left: 30px;
    margin-right: 10px;
  }
}

.sidebar {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  width: 250px;
  background-color: #333;
  color: #fff;
  overflow-y: auto;
  padding: 20px;
  transition: transform 0.3s ease-in-out;
  transform: translateX(-250px);
}

.sidebar--open {
  transform: translateX(0);
}

.container.sidebar-open .sidebar {
  transform: translateX(0);
}

.container.sidebar-open .main {
  margin-left: 250px;
}

.sidebar__toggle {
  position: absolute;
  top: 10px;
  right: 10px;
  font-size: 20px;
  background-color: transparent;
  border: none;
  color: white;
  cursor: pointer;
}

.sidebar--open .sidebar__toggle::before {
  content: "\00d7";
}

.sidebar__toggle::before {
  content: "\2261";
  display: block;
  padding-top: px;
}

audio {
  width: 100%;
  margin-bottom: 20px;
}

.showMore {
  font-size: 24px;
  padding: 10px 20px;
  border: none;
  border-radius: 20px;
  background-color: #1db954;
  color: #fff;
  cursor: pointer;
  margin-bottom: 40px;
  outline: none;
  display: block;
  margin: 0 auto;
}

.add-to-favorites {
  font-size: 24px;
  padding: 10px 20px;
  border: none;
  border-radius: 20px;
  background-color: #1db954;
  color: #fff;
  cursor: pointer;
  margin-bottom: 20px;
  outline: none;
}

.add-to-playlist {
  font-size: 24px;
  padding: 10px 20px;
  border: none;
  border-radius: 20px;
  background-color: #1db954;
  color: #fff;
  cursor: pointer;
  margin-bottom: 40px;
  outline: none;
}

.search-btn:hover {
  background-color: #18ac38;
  color: white;
}

.add-to-favorites:active {
  transform: scale(0.95);
}

.add-to-playlist:active {
  transform: translateY(2px);
}

.genre-link {
color: rgb(255, 255, 255);
}

.genre-link:hover {
  padding: 10px 10px;
  border: none;
  border-radius: 20px;
  background-color: #1db954;
  color: #fff;
  cursor: pointer;
  margin-bottom: 40px;
  outline: none;
}

.genre-link.active {
  padding: 10px 10px;
  border: none;
  border-radius: 20px;
  background-color: #1db954;
  color: #fff;
  cursor: pointer;
  margin-bottom: 40px;
  outline: none;
}


h1 {
  font-weight: bold;
}

h2 {
  font-size: 24px;
  margin-bottom: 20px;
}

ul {
  list-style: none;
  margin: 0;
  padding: 0;
}

li {
  font-size: 18px;
  margin-bottom: 10px;
}
</style>