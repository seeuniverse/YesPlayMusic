<template>
  <div class="search" v-show="show">
    <div class="row" v-show="artists.length > 0 || albums.length > 0">
      <div class="artists">
        <div class="section-title" v-show="artists.length > 0"
          >{{ $t("search.artist")
          }}<router-link :to="`/search/${keywords}/artists`">{{
            $t("home.seeMore")
          }}</router-link></div
        >
        <CoverRow
          type="artist"
          :columnNumber="3"
          :items="artists.slice(0, 3)"
          gap="34px 24px"
          :withoutPadding="true"
        />
      </div>

      <div class="albums">
        <div class="section-title" v-show="albums.length > 0"
          >{{ $t("search.album")
          }}<router-link :to="`/search/${keywords}/albums`">{{
            $t("home.seeMore")
          }}</router-link></div
        >
        <CoverRow
          type="album"
          :items="albums.slice(0, 3)"
          subText="artist"
          :columnNumber="3"
          subTextFontSize="14px"
          gap="34px 24px"
          :withoutPadding="true"
        />
      </div>
    </div>

    <div class="tracks" v-show="tracks.length > 0">
      <div class="section-title"
        >{{ $t("search.song")
        }}<router-link :to="`/search/${keywords}/tracks`">{{
          $t("home.seeMore")
        }}</router-link></div
      >
      <TrackList :tracks="tracks" type="tracklist" :withoutPadding="true" />
    </div>

    <div class="music-videos" v-show="musicVideos.length > 0">
      <div class="section-title"
        >{{ $t("search.mv")
        }}<router-link :to="`/search/${keywords}/music-videos`">{{
          $t("home.seeMore")
        }}</router-link></div
      >
      <MvRow :mvs="musicVideos.slice(0, 5)" :withoutPadding="true" />
    </div>

    <div class="playlists" v-show="playlists.length > 0">
      <div class="section-title"
        >{{ $t("search.playlist")
        }}<router-link :to="`/search/${keywords}/playlists`">{{
          $t("home.seeMore")
        }}</router-link></div
      >
      <CoverRow
        type="playlist"
        :items="playlists.slice(0, 12)"
        subText="title"
        :columnNumber="6"
        subTextFontSize="14px"
        gap="34px 24px"
        :withoutPadding="true"
      />
    </div>

    <div class="no-results" v-show="!haveResult">
      <div
        ><svg-icon icon-class="search" />
        {{
          keywords.length === 0 ? "输入关键字搜索" : $t("search.noResult")
        }}</div
      >
    </div>
  </div>
</template>

<script>
import { getTrackDetail } from "@/api/track";
import { search } from "@/api/others";
import NProgress from "nprogress";

import TrackList from "@/components/TrackList.vue";
import MvRow from "@/components/MvRow.vue";
import CoverRow from "@/components/CoverRow.vue";

export default {
  name: "Search",
  components: {
    TrackList,
    MvRow,
    CoverRow,
  },
  data() {
    return {
      show: false,
      tracks: [],
      artists: [],
      albums: [],
      playlists: [],
      musicVideos: [],
    };
  },
  computed: {
    keywords() {
      return this.$route.params.keywords ?? "";
    },
    haveResult() {
      return (
        this.tracks.length +
          this.artists.length +
          this.albums.length +
          this.playlists.length +
          this.musicVideos.length >
        0
      );
    },
  },
  methods: {
    playTrackInSearchResult(id) {
      let track = this.tracks.find((t) => t.id === id);
      this.$store.state.player.appendTrackToPlayerList(track, true);
    },
    search(type = "all") {
      const typeTable = {
        all: 1018,
        musicVideos: 1004,
        tracks: 1,
        albums: 10,
        artists: 100,
        playlists: 1000,
      };
      return search({
        keywords: this.keywords,
        type: typeTable[type],
        limit: 16,
      }).then((result) => {
        return { result: result.result, type };
      });
    },
    getData() {
      NProgress.start();
      this.show = false;

      const requestAll = (requests) => {
        const keywords = this.keywords;
        Promise.all(requests).then((results) => {
          if (keywords != this.keywords) return;
          results.map((result) => {
            const searchType = result.type;
            if (result.result === undefined) return;
            result = result.result;
            switch (searchType) {
              case "all":
                this.result = result;
                break;
              case "musicVideos":
                this.musicVideos = result.mvs ?? [];
                break;
              case "artists":
                this.artists = result.artists ?? [];
                break;
              case "albums":
                this.albums = result.albums ?? [];
                break;
              case "tracks":
                this.tracks = result.songs ?? [];
                this.getTracksDetail();
                break;
              case "playlists":
                this.playlists = result.playlists ?? [];
                break;
            }
          });
          NProgress.done();
          this.show = true;
        });
      };

      const requests = [
        this.search("artists"),
        this.search("albums"),
        this.search("tracks"),
      ];
      const requests2 = [this.search("musicVideos"), this.search("playlists")];

      requestAll(requests);
      requestAll(requests2);
    },
    getTracksDetail() {
      const trackIDs = this.tracks.map((t) => t.id);
      if (trackIDs.length === 0) return;
      getTrackDetail(trackIDs.join(",")).then((result) => {
        this.tracks = result.songs;
      });
    },
  },
  created() {
    this.getData();
  },
  watch: {
    keywords: function (newKeywords) {
      if (newKeywords.length === 0) return;
      this.getData();
    },
  },
};
</script>

<style lang="scss" scoped>
.search {
  padding: var(--main-content-padding);
}

.section-title {
  font-weight: 600;
  font-size: 22px;
  opacity: 0.88;
  color: var(--color-text);
  margin-bottom: 16px;

  display: flex;
  justify-content: space-between;
  align-items: center;
  a {
    font-size: 13px;
    font-weight: 600;
    opacity: 0.68;
  }
}

.row {
  display: flex;
  flex-wrap: wrap;
  margin-top: 98px;

  .artists {
    flex: 1;
    margin-right: 8rem;
  }
  .albums {
    flex: 1;
  }
}

.tracks,
.music-videos,
.playlists {
  margin-top: 46px;
}

.no-results {
  position: absolute;
  top: 64px;
  right: 0;
  left: 0;
  bottom: 64px;
  font-size: 24px;
  color: var(--color-text);
  opacity: 0.38;
  display: flex;
  justify-content: center;
  align-items: center;
  div {
    display: flex;
    align-items: center;
  }
  .svg-icon {
    height: 24px;
    width: 24px;
    margin-right: 16px;
  }
}
</style>
