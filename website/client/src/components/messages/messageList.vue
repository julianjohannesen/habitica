<template>
  <perfect-scrollbar
    ref="container"
    class="container-fluid"
    :class="{'disable-perfect-scroll': disablePerfectScroll}"
    :options="psOptions"
  >
    <div class="row loadmore">
      <div v-if="canLoadMore && !isLoading">
        <div class="loadmore-divider"></div>
        <button
          class="btn btn-secondary"
          @click="triggerLoad()"
        >
          {{ $t('loadEarlierMessages') }}
        </button>
        <div class="loadmore-divider"></div>
      </div>
      <h2
        v-show="isLoading"
        class="col-12 loading"
      >
        {{ $t('loading') }}
      </h2>
    </div>
    <div
      v-for="(msg) in messages"
      :key="msg.id"
      class="row message-row"
      :class="{ 'margin-right': user._id !== msg.uuid}"
    >
      <div
        v-if="user._id !== msg.uuid"
        class="d-flex flex-grow-1"
      >
        <avatar
          v-if="msg.userStyles || (cachedProfileData[msg.uuid]
            && !cachedProfileData[msg.uuid].rejected)"
          class="avatar-left"
          :member="msg.userStyles || cachedProfileData[msg.uuid]"
          :avatar-only="true"
          :override-top-padding="'14px'"
          :hide-class-badge="true"
          @click.native="showMemberModal(msg.uuid)"
        />
        <div class="card card-right">
          <message-card
            :msg="msg"
            @message-removed="messageRemoved"
            @show-member-modal="showMemberModal"
            @message-card-mounted="itemWasMounted"
          />
        </div>
      </div>
      <div
        v-if="user._id === msg.uuid"
        class="d-flex flex-grow-1"
      >
        <div class="card card-left">
          <message-card
            :msg="msg"
            @message-removed="messageRemoved"
            @show-member-modal="showMemberModal"
            @message-card-mounted="itemWasMounted"
          />
        </div>
        <avatar
          v-if="msg.userStyles
            || (cachedProfileData[msg.uuid] && !cachedProfileData[msg.uuid].rejected)"
          class="avatar-right"
          :member="msg.userStyles || cachedProfileData[msg.uuid]"
          :avatar-only="true"
          :hide-class-badge="true"
          :override-top-padding="'14px'"
          @click.native="showMemberModal(msg.uuid)"
        />
      </div>
    </div>
  </perfect-scrollbar>
</template>

<style lang="scss" scoped>
  @import '~@/assets/scss/colors.scss';
  @import '~vue2-perfect-scrollbar/dist/vue2-perfect-scrollbar.css';

  .disable-perfect-scroll {
    overflow-y: inherit !important;
  }

  .avatar {
    width: 15%;
    min-width: 8rem;
    height: 120px;
    padding-top: 0 !important;
  }

  .avatar-left {
    margin-left: -1rem;
  }

  .avatar-right {
    margin-left: -1rem;
  }

  .card {
    border: 0px;
    margin-bottom: 1rem;
    padding: 0rem;
    width: 684px;
  }
  .message-row {
    margin-left: 12px;
    margin-right: 12px;

    &:not(.margin-right) {
      .d-flex {
        justify-content: flex-end;
      }
    }
  }
  @media only screen and (max-width: 1200px) {
    .card {
      width: 100%;
    }
  }

  @media only screen and (min-width: 1400px) {
    .message-row {
      margin-left: -15px;
      margin-right: -30px;
    }
  }

  .card-left {
    border: 1px solid $purple-500;
  }

  .card-right {
    border: 1px solid $gray-500;
  }

  .hr {
    width: 100%;
    height: 20px;
    border-bottom: 1px solid $gray-500;
    text-align: center;
    margin: 2em 0;
  }

  .hr-middle {
    font-size: 16px;
    font-weight: bold;
    font-family: 'Roboto Condensed';
    line-height: 1.5;
    text-align: center;
    color: $gray-200;
    background-color: $gray-700;
    padding: .2em;
    margin-top: .2em;
    display: inline-block;
    width: 100px;
  }

  .loadmore {
    justify-content: center;
    margin-right: 12px;

    > div {
      display: flex;
      width: 100%;
      align-items: center;

      button {
        text-align: center;
        color: $gray-50;
        margin-top: 12px;
        margin-bottom: 24px;
      }
    }
  }

  .loadmore-divider {
    height: 1px;
    background-color: $gray-500;
    flex: 1;
    margin-left: 24px;
    margin-right: 24px;

    &:last-of-type {
      margin-right: 0;
    }
  }

  .loading {
    padding-left: 1.5rem;
    margin-bottom: 1rem;
  }


</style>

<script>
import moment from 'moment';
import axios from 'axios';
import debounce from 'lodash/debounce';
import { PerfectScrollbar } from 'vue2-perfect-scrollbar';
import { mapState } from '@/libs/store';

import Avatar from '../avatar';
import messageCard from './messageCard';

export default {
  components: {
    Avatar,
    messageCard,
    PerfectScrollbar,
  },
  props: {
    chat: {},
    isLoading: Boolean,
    canLoadMore: Boolean,
  },
  data () {
    return {
      currentDayDividerDisplay: moment().day(),
      cachedProfileData: {},
      currentProfileLoadedCount: 0,
      currentProfileLoadedEnd: 10,
      loading: false,
      handleScrollBack: false,
      lastOffset: -1,
      disablePerfectScroll: false,
    };
  },
  mounted () {
    this.loadProfileCache();

    this.$el.addEventListener('selectstart', () => this.handleSelectStart());
    this.$el.addEventListener('mouseup', () => this.handleSelectChange());
  },
  created () {
    window.addEventListener('scroll', this.handleScroll);
  },
  beforeDestroy () {
    this.$el.removeEventListener('selectstart', () => this.handleSelectStart());
    this.$el.removeEventListener('mouseup', () => this.handleSelectChange());
  },
  destroyed () {
    window.removeEventListener('scroll', this.handleScroll);
  },
  computed: {
    ...mapState({ user: 'user.data' }),
    // @TODO: We need a different lazy load mechnism.
    // But honestly, adding a paging route to chat would solve this
    messages () {
      this.loadProfileCache();
      return this.chat;
    },
    psOptions () {
      return {
        suppressScrollX: true,
      };
    },
  },
  methods: {
    handleScroll () {
      this.loadProfileCache(window.scrollY / 1000);
    },
    async triggerLoad () {
      const container = this.$refs.container.$el;

      // get current offset
      this.lastOffset = container.scrollTop - (container.scrollHeight - container.clientHeight);
      // disable scroll
      // container.style.overflowY = 'hidden';

      const canLoadMore = !this.isLoading && this.canLoadMore;

      if (canLoadMore) {
        const triggerLoadResult = this.$emit('triggerLoad');

        await triggerLoadResult;

        this.handleScrollBack = true;
      }
    },
    loadProfileCache: debounce(function loadProfileCache (screenPosition) {
      this._loadProfileCache(screenPosition);
    }, 1000),
    async _loadProfileCache (screenPosition) {
      if (this.loading) return;
      this.loading = true;

      const promises = [];
      const noProfilesLoaded = Object.keys(this.cachedProfileData).length === 0;

      // @TODO: write an explination
      // @TODO: Remove this after enough messages are cached
      if (!noProfilesLoaded && screenPosition
        && Math.floor(screenPosition) + 1 > this.currentProfileLoadedEnd / 10) {
        this.currentProfileLoadedEnd = 10 * (Math.floor(screenPosition) + 1);
      } else if (!noProfilesLoaded && screenPosition) {
        return;
      }

      const aboutToCache = {};
      this.messages.forEach(message => {
        const { uuid } = message;

        if (message.userStyles) {
          this.$set(this.cachedProfileData, uuid, message.userStyles);
        }

        if (Boolean(uuid) && !this.cachedProfileData[uuid] && !aboutToCache[uuid]) {
          if (uuid === 'system' || this.currentProfileLoadedCount === this.currentProfileLoadedEnd) return;
          aboutToCache[uuid] = {};
          promises.push(axios.get(`/api/v4/members/${uuid}`));
          this.currentProfileLoadedCount += 1;
        }
      });

      const results = await Promise.all(promises);
      results.forEach(result => {
        // We could not load the user. Maybe they were deleted.
        // So, let's cache empty so we don't try again
        if (!result || !result.data || result.status >= 400) {
          return;
        }

        const userData = result.data.data;
        this.$set(this.cachedProfileData, userData._id, userData);
      });

      // Merge in any attempts that were rejected so we don't attempt again
      for (const uuid in aboutToCache) {
        if (!this.cachedProfileData[uuid]) {
          this.$set(this.cachedProfileData, uuid, { rejected: true });
        }
      }

      this.loading = false;
    },
    displayDivider (message) {
      if (this.currentDayDividerDisplay !== moment(message.timestamp).day()) {
        this.currentDayDividerDisplay = moment(message.timestamp).day();
        return true;
      }

      return false;
    },
    async showMemberModal (memberId) {
      let profile = this.cachedProfileData[memberId];

      if (!profile._id) {
        const result = await this.$store.dispatch('members:fetchMember', { memberId });
        if (result.response && result.response.status === 404) {
          return this.$store.dispatch('snackbars:add', {
            title: 'Habitica',
            text: this.$t('messageDeletedUser'),
            type: 'error',
            timeout: false,
          });
        }
        this.cachedProfileData[memberId] = result.data.data;
        profile = result.data.data;
      }

      // Open the modal only if the data is available
      if (profile && !profile.rejected) {
        this.$router.push({ name: 'userProfile', params: { userId: profile._id } });
      }

      return null;
    },
    itemWasMounted: debounce(function itemWasMounted () {
      if (this.handleScrollBack) {
        this.handleScrollBack = false;

        const container = this.$refs.container.$el;
        const offset = container.scrollHeight - container.clientHeight;

        const newOffset = offset + this.lastOffset;

        container.scrollTo(0, newOffset);
        // enable scroll again
        // container.style.overflowY = 'scroll';
      }
    }, 50),
    messageRemoved (message) {
      this.$emit('message-removed', message);
    },
    handleSelectStart () {
      this.disablePerfectScroll = true;
    },
    handleSelectChange () {
      this.disablePerfectScroll = false;
    },
  },
};
</script>
