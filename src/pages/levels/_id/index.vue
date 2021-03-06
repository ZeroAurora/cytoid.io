<template lang="pug">
  .section: .container(v-if="level" style="margin-top: 256px;")
    captcha(ref="captcha")
    h1.is-size-1(v-text="level.title")
    h5.is-size-5(
      style="color: rgba(255, 255, 255, 0.9); margin-bottom: 20px;"
      v-if="level.metadata.artist"
      v-text="level.metadata.artist.name"
    )
    div(style="margin-bottom: 48px;")
      difficulty-badge.ele3(v-for="chart in level.charts" :key="chart.id" :value="chart" style="margin-right: 8px;")
    .buttons(style="margin-bottom: 32px;")
      b-button(
        size="is-large"
        type="is-download"
        :loading="downloadBtnLoading"
        @click="download"
        icon-left="download"
      ) {{ $t('download_btn', { size: formatSize(level.size)})}}
      template(v-if="$store.state.user")
        b-button(
          v-if="level.owned === null"
          size="is-large"
          type="is-secondary"
          :loading="addToLibraryBtnLoading"
          @click="addToLibrary"
          icon-pack="fal"
          icon-left="heart"
        ) {{ $t('favorite_verb') }}
        b-button(
          v-else-if="level.owned === false"
          size="is-large"
          type="is-danger"
          :loading="addToLibraryBtnLoading"
          @click="removeFromLibrary"
          icon-left="heart"
        ) {{ $t('favorite_pass_tense') }}
        b-button(
          v-else-if="level.owned === true"
          size="is-large"
          type="is-danger"
          disabled
          icon-left="album"
        ) Owned
      nuxt-link.button.is-large(
        :to="{ name: 'levels-id-manage', params: { id: level.uid }}"
        v-if="$store.state.user && (level.owner.id === $store.state.user.id || $store.state.user.role === 'admin' || $store.state.user.role === 'moderator')"
      )
          b-icon(icon="briefcase")
          span(v-t="'manage_btn'")
    .notification.is-warning(v-if="level.state === 'PRIVATE'" v-t="'message_private'")
    .notification.is-primary(v-if="level.state === 'UNLISTED'" v-t="'message_unlisted'")
    .notification.is-danger(v-if="level.censored" v-t="{ path: 'message_censored', args: { reason: level.censored } }")
    .columns
      .column.is-one-third
        .box
          player-avatar(style="margin-bottom: 16px;" :player="level.owner")
          .content(style="overflow: auto;" v-if="levelDescription" v-html="levelDescription")
          p.box-subtitle(v-t="'details_card_rating_title'")
          b-rate(
            icon-pack="fas"
            :value="(level.rating.rating || level.rating.average) * 0.5"
            :disabled="!$store.state.user"
            @change="rate"
            :custom-text="`${(Math.floor(level.rating.average * 0.5 * 100) / 100).toFixed(2)} (${level.rating.total})`"
          )
          template(v-if="level.tags.length > 0")
            .box-subtitle(v-t="'details_card_tags_title'")
            .tags
              a.tag(v-for="tag in level.tags" :key="tag" :href="'/levels?tags=' + tag.toLowerCase()" v-text="tag")
          .box-subtitle(v-t="'details_card_last_updated_title'")
          p {{$dateFormatCalendar(level.modificationDate)}}, {{ $dateFromNow(level.modificationDate) }}
        meta-box(:metadata="level.metadata")
      .column.is-two-thirds
        .box.rankings-card.is-gradient(:style="rankingsHeaderGradient")
          .box-subtitle(v-t="'difficulty_card_title'")
          .tabs.is-small.is-toggle: ul
            li(
              v-for="chart in level.charts"
              :key="chart.id"
              :class="{ 'is-active': rankingsChartType === chart.type }"
              @click="rankingsChartType = chart.type"
            )
              a {{ chart.name || convertedDifficultyName(chart.type)}}
          b-table.leaderboard-table(
            :data="leaderboard"
            :mobile-cards="false"
            centered
            scrollable
            hoverable
            paginated
            backend-pagination
            :current-page="rankingsPagination.currentPage + 1"
            :loading="rankingsLoading"
            :per-page="rankingsPagination.recordsPerPage"
            :total="rankingsPagination.total"
            :row-class="rowClass"
            @page-change="pageChange"
          )
            template(slot="empty")
              empty-placeholder
            b-table-column(v-slot="props" field="id" label="rank") \#{{ props.index + 1 + rankingsPagination.currentPage * rankingsPagination.recordsPerPage }}
            b-table-column(v-slot="props" field="owner" label="player")
              nuxt-link.row-score-avatar(:to="{name: 'profile-id', params: { id: props.row.owner.uid || props.row.owner.id }}")
                avatar(:source="props.row.owner.avatar.small" fixed)
                span(v-text="props.row.owner.name || props.row.owner.uid")
            b-table-column(v-slot="props" field="score" label="score")
              score-badge(:value="props.row.score")
              span(style="margin-left: 4px;" v-text="props.row.score")
            b-table-column(v-slot="props" field="accuracy" label="acc") {{ (Math.floor(props.row.accuracy * 100 * 100) / 100) }}%
            b-table-column(v-slot="props" field="maxCombo" label="max combo") {{ props.row.details.maxCombo ? (props.row.details.maxCombo + 'x') : 'Unknown' }}
            b-table-column(v-slot="props" field="mods" label="mods")
              span(v-if="props.row.mods.length === 0 || props.row.mods[0] === ''") N/A
              span(v-else)
                img(
                  v-for="mod in props.row.mods"
                  :key="mod"
                  :title="getModById(mod).title"
                  :src="getModById(mod).src"
                  style="height: 20px; padding-bottom: 2px; max-width: unset; margin-right: 4px;"
                )
            b-table-column(v-slot="props" field="perfect" label="perfect") {{ props.row.details.perfect }}
            b-table-column(v-slot="props" field="great" label="great") {{ props.row.details.great }}
            b-table-column(v-slot="props" field="good" label="good") {{ props.row.details.good }}
            b-table-column(v-slot="props" field="bad" label="bad") {{ props.row.details.bad }}
            b-table-column(v-slot="props" field="miss" label="miss") {{ props.row.details.miss }}
            b-table-column(v-slot="props" field="date" label="achieved") {{ $dateFromNow(props.row.date) }}
        thread(category="level" :thread="level.uid")
    .play-button-container
      play-button(:src="level.bundle.musicPreview")
</template>

<script>
import Captcha from '@/components/Captcha'
import marked from 'marked'
import Thread from '@/components/comments/Thread'
import ScoreBadge from '@/components/level/ScoreBadge'
import DifficultyBadge from '@/components/level/DifficultyBadge'
import PlayButton from '@/components/level/PlayButton'
import PlayerAvatar from '@/components/player/PlayerAvatar'
import { formatBytes, Meta } from '@/utils'
import gql from 'graphql-tag'
import { handleErrorBlock } from '@/plugins/antd'
import MetaBox from '@/components/MetaBox'
import EmptyPlaceholder from '@/components/EmptyPlaceholder'
import DownloadModal from '@/components/level/DownloadModal'

const ModIconKeys = [
  'ap',
  'auto',
  'autodrag',
  'autoflick',
  'autohold',
  'exhard',
  'fast',
  'fc',
  'flipall',
  'flipx',
  'flipy',
  'hard',
  'hidenotes',
  'hidescanline',
  'slow'
]
const ModIconKeyPathMap = {}
for (const key of ModIconKeys) {
  ModIconKeyPathMap[key] = require(`@/assets/icons/${key}.png`)
}
// eslint-disable-next-line no-unused-vars
const modNames = {
  hidenotes: 'Invisible',
  hidescanline: 'No Scanline',
  slow: 'Slow',
  fast: 'Fast',
  hard: 'HYPER',
  exhard: 'ANOTHER',
  ap: 'All Perfect',
  fc: 'Full Combo',
  flipall: 'Flip All',
  flipx: 'Flip X',
  flipy: 'Flip Y',
}
const query = gql`query FetchLevel($uid: String!){
  level(uid: $uid) {
    id
    uid
    title
    description
    state
    censored
    size
    tags
    creationDate
    modificationDate
    owned
    owner {
      id
      uid
      name
      avatar { large }
    }
    charts {
      difficulty
      type
      name
      notesCount
    }
    metadata {
      artist {
        name
        localized_name
        url
      }
      illustrator {
        name
        localized_name
        url
      }
      charter {
        name
        localized_name
        url
      }
      storyboarder {
        name
        localized_name
        url
      }
    }
    rating {
      average
      total
      distribution
      rating
    }
    bundle {
      musicPreview
      backgroundImage {
        original
      }
    }
  }
}`
const rankingQuery = gql`query FetchLevelRanking($levelUid: String!, $type: String!, $start: Int!) {
  chart(levelUid: $levelUid, chartType: $type) {
    id
    numPlayers
    leaderboard(limit: 10, start: $start) {
      id
      date
      owner {
        id
        uid
        name
        avatar {
          small
        }
      }
      score
      accuracy
      mods
      details {
        perfect
        great
        good
        bad
        miss
        maxCombo
      }
    }
  }
}
`
export default {
  layout: 'background',
  name: 'LevelDetails',
  components: {
    ScoreBadge,
    PlayerAvatar,
    DifficultyBadge,
    PlayButton,
    Thread,
    MetaBox,
    Captcha,
    EmptyPlaceholder
  },
  async asyncData ({ app, params, error, store }) {
    const level = await app.apolloProvider.defaultClient.query({
      query,
      variables: { uid: params.id }
    }).then(({ data }) => data?.level)
      .catch(err => handleErrorBlock(err, error))
    if (!level) {
      return error({ statusCode: 404, message: 'Level not found' })
    }
    const image = level.bundle?.backgroundImage?.original
    if (image) {
      store.commit('setBackground', { source: image })
    }
    const defaultType = (level.charts && level.charts.length > 0) ? level.charts[0].type : null
    const result = {
      level,
      rankingsChartType: defaultType,
    }
    if (defaultType) {
      const chart = await app.apolloProvider.defaultClient.query({
        query: rankingQuery,
        variables: { levelUid: params.id, type: defaultType, start: 0 },
      }).then(a => a?.data?.chart)
      if (chart?.numPlayers > 0) {
        result.rankingsPagination = {
          total: chart.numPlayers,
          currentPage: 0,
          recordsPerPage: 10,
        }
        result.leaderboard = chart.leaderboard
      }
    }
    return result
  },
  data: () => ({
    level: null,
    downloadBtnLoading: false,
    addToLibraryBtnLoading: false,
    leaderboard: [],
    rankingsChartType: null,
    rankingsPagination: {
      currentPage: 0,
      total: 0,
      recordsPerPage: 10,
    },
    rankingsLoading: false,
  }),
  computed: {
    levelDescription () {
      return this.level.description !== null ? marked(this.level.description) : null
    },
    rankingsHeaderGradient () {
      return {
        '--box-background-gradient': {
          extreme: 'linear-gradient(to bottom right, #6f0000, #200122)',
          easy: 'linear-gradient(to top left, #4ca2cd, #67B26F)',
          hard: 'linear-gradient(to top left, #B06AB3, #4568DC)'
        }[this.rankingsChartType]
      }
    },
  },
  watch: {
    rankingsChartType () {
      this.rankingsPagination.currentPage = 0
      this.reloadLeaderboard()
    }
  },
  methods: {
    addToLibrary () {
      this.addToLibraryBtnLoading = true
      this.$apollo.mutate({
        mutation: gql`mutation AddToLibrary($levelId: Int!) {
          addToLibrary(levelId: $levelId)
        }`,
        variables: {
          levelId: this.level.id,
        }
      })
        .then(() => {
          this.level.owned = false
          this.$buefy.toast.open({
            message: 'Level added to Library',
            type: 'is-success',
          })
        })
        .catch(err => this.handleErrorToast(err))
        .then(() => {
          this.addToLibraryBtnLoading = false
        })
    },
    removeFromLibrary () {
      this.$buefy.dialog.confirm({
        title: `Remove ${this.level.title} from favorites`,
        message: `Are you sure you want to remove ${this.level.title} from your library?`,
        confirmText: 'Remove',
        type: 'is-danger',
        hasIcon: true,
        onConfirm: () => {
          this.addToLibraryBtnLoading = true
          this.$apollo.mutate({
            mutation: gql`mutation RemoveFromLibrary($levelId: Int!) {
              removeFromLibrary(levelId: $levelId)
            }`,
            variables: {
              levelId: this.level.id,
            }
          })
            .then(() => {
              this.level.owned = null
              this.$buefy.toast.open({
                message: 'Level removed from favorites',
                type: 'is-warning',
              })
            })
            .catch(err => this.handleErrorToast(err))
            .then(() => {
              this.addToLibraryBtnLoading = false
            })
        }
      })
    },
    pageChange (pageNum) {
      this.rankingsPagination.currentPage = pageNum - 1
      this.reloadLeaderboard()
    },
    async reloadLeaderboard () {
      this.rankingsLoading = true
      const chart = await this.$apollo.query({
        query: rankingQuery,
        variables: {
          levelUid: this.$route.params.id,
          type: this.rankingsChartType,
          start: this.rankingsPagination.currentPage * this.rankingsPagination.recordsPerPage
        },
      }).then(a => a?.data?.chart)
      this.leaderboard = chart?.leaderboard
      if (chart?.numPlayers > 0) {
        this.rankingsPagination.total = chart.numPlayers
      } else {
        this.rankingsPagination.total = 0
      }
      this.rankingsLoading = false
    },
    rate (rating) {
      rating *= 2
      this.$apollo.mutate({
        mutation: gql`mutation LevelRate($uid: String!, $rating: Int) {
          rateLevel(id: $uid, rating: $rating) {
            average
            total
            distribution
            rating
          }
        }`,
        variables: {
          uid: this.level.uid,
          rating,
        },
      })
        .then((res) => {
          this.level.rating = res.data.rateLevel
        })
    },
    getModById (id) {
      id = id.toLowerCase()
      return {
        title: modNames[id],
        src: ModIconKeyPathMap[id],
      }
    },
    rowClass (record, index) {
      let classes = 'row-score'
      if (record.score === 1000000) {
        classes += ' row-score-max'
      } else if (record.score >= 999500) {
        classes += ' row-score-sss'
      }
      if (this.rankingsPagination.currentPage === 0) {
        if (index === 0) {
          classes += ' row-score-1st'
        } else if (index === 1) {
          classes += ' row-score-2nd'
        } else if (index === 2) {
          classes += ' row-score-3rd'
        }
      }
      return classes
    },
    formatSize (size) {
      return formatBytes(size)
    },
    download () {
      if (this.$store.state.user) {
        this.downloadBtnLoading = true
        this.$refs.captcha.execute()
          .then((code) => {
            this.$axios
              .post(`/levels/${this.level.uid}/resources`, { captcha: code })
              .then((res) => {
                this.downloadBtnLoading = false
                this.$buefy.modal.open({
                  parent: this,
                  component: DownloadModal,
                  hasModalCard: true,
                  trapFocus: true,
                  props: {
                    level: this.level,
                    url: res.data.package,
                  }
                })
              })
            global.window.gtag('event', 'download', {
              event_category: 'levels',
              event_label: 'succeed',
              value: this.level.uid
            })
          })
      } else {
        global.window.gtag('event', 'download', {
          event_category: 'levels',
          event_label: 'rejected-login',
          value: this.level.uid
        })
        this.$router.push({ name: 'session-login', query: { origin: `/levels/${this.$route.params.id}` } })
      }
    },
    convertedDifficultyName (name) {
      return {
        easy: 'Easy',
        hard: 'Hard',
        extreme: 'Extreme',
      }[name]
    },
  },
  head () {
    const meta = new Meta(this.level?.title || 'Level', this.level?.description || '')
    meta.extend('author', this.level?.owner?.name || this.level?.owner?.uid)
    meta.extend('og:image', this.level?.bundle?.background)
    return meta
  },
  i18n: {
    key: 'level_details'
  }
}
</script>

<style lang="scss">
.leaderboard-table {
  .table-wrapper {
    overflow: scroll;
    .table {
      border: none !important;
      border-collapse: collapse !important;
      white-space: nowrap;
    }
  }
  .cytoid-avatar {
    height: 20px;
    width: 20px;
    margin-right: .5rem;
  }
  .row-score {
    .row-score-avatar {
      color: white;
      display: flex;
      overflow: hidden;
      img {
        flex-shrink: 0;
      }
    }
    &.row-score-sss {
      background-image: linear-gradient(315deg, #fc9842 0%, #fe5f75 100%);
      font-weight: bold;
    }
    &.row-score-max {
      background-image: linear-gradient(19deg, #21D4FD 0%, #B721FF 100%);
      font-weight: bold;
    }

    &.row-score-1st {
      .cytoid-avatar {
        height: 28px;
        width: 28px;
      }
      font-size: 20px;
    }

    &.row-score-2nd {
      .cytoid-avatar {
        height: 24px;
        width: 24px;
      }
      font-size: 18px;
    }

    &.row-score-3rd {
      font-size: 16px;
      .cytoid-avatar {
        height: 22px;
        width: 22px;
      }
    }

    // Tell Safari to fix their shit
    // See https://bug.webkit.org/show_bug.cgi?id=9268
    @media not all and (min-resolution: .001dpcm) {
      @supports (-webkit-appearance: none) {
        &.row-score-sss {
          color: white;
          background-image: linear-gradient(315deg, #FD7C5C 0%, #FD7C5C 100%);
        }

        &.row-score-max {
          color: white;
          background-image: linear-gradient(19deg, #6C7BFE 0%, #6C7BFE 100%);
        }
      }
    }
  }
}
$play-button-container-size: 56px;
.play-button-container {
  position: fixed;
  z-index: 64;
  width: $play-button-container-size;
  height: $play-button-container-size;
  bottom: 20px;
  right: 20px;
  padding: 0;
  margin: 0;
  color: #FFF !important;
  border-radius: 50%;
  background-image: linear-gradient(to right, #DD2476, #FF512F) !important;
  box-shadow: rgba(221, 36, 118, 0.3) 0px 3px 5px -1px, rgba(221, 36, 118, 0.14) 0px 6px 10px 0px, rgba(221, 36, 118, 0.12) 0px 1px 18px 0px;
  display: flex;
  .play-button {
    margin: auto;
  }
}
</style>
