<template>
  <div class="fullwidth-box" :class="classes">
    <div class="info-text">
      <h2 class="title is-3">
        {{ $t('auth.settings.invites', 'Invites') }}
      </h2>
      <p class="subtitle is-6">{{ $t('auth.settings.invitesDescription') }}</p>
    </div>
    <div class="ticket-wrapper">
      <div class="columns is-multiline" v-if="invites.data.length">
        <invite-ticket v-for="invite in invites.data" :key="invite._id"
                      :code="invite.code"
                      :link="copyInviteLink(invite)"
                      :used="invite.wasUsed"/>
      </div>
      <div v-else class="has-text-centered" style="padding: 3rem">
        <p>{{ $t('auth.settings.invitesEmpty', 'click below') }}</p>
      </div>
    </div>
    <footer class="card-footer">
      <hc-button :isLoading="isLoading"
                  :disabled="isLoading || !hasInvitesLeft"
                  @click.prevent="save">
        <hc-icon icon="ticket" class="icon-left" /> {{ $t('auth.settings.invitesGenerateButton', 'Generate invite codes', {}, settings.invites.maxInvitesByUser) }}
      </hc-button>
    </footer>
  </div>
</template>

<script>
  import { mapGetters } from "vuex";
  import animatable from '~/components/mixins/animatable'
  import InviteTicket from '~/components/Auth/InviteTicket'
  import urlHelper from '~/helpers/urls'

  export default {
    mixins: [animatable],
    components: {
      'invite-ticket': InviteTicket
    },
    async asyncData ({app, store, redirect}) {
      if (!store.getters['settings/showInvites']) {
        redirect('/auth/settings')
      }
      const res = await app.$api.service('user-invites').find()
      return {
        invites: res || { total: 0, data: [] }
      }
    },
    data() {
      return {
        isLoading: false,
        invites: { total: 0, data: [] },
        fetchTimer: null,
        refrehInterval: 20000
      };
    },
    mounted () {
      this.fetchTimer = setTimeout(this.fetch, this.refrehInterval)
    },
    beforeDestroy () {
      clearTimeout(this.fetchTimer);
    },
    watch: {
      showInvites (show) {
        if (!show) {
          this.$router.replace('/auth/settings')
        }
      }
    },
    methods: {
      async fetch () {
        clearTimeout(this.fetchTimer);

        let res
        try {
          res = await this.$api.service('user-invites').find()
        } catch (err) {
          console.log(err.message)
        }
        this.invites = res || this.invites || { total: 0, data: [] }
        // this.fetchTimer = setTimeout(this.fetch, this.refrehInterval)
      },
      copyInviteLink (invite) {
        if (process.client) {
          return `${location.origin}/auth/register?email=${invite.email}&code=${invite.code}&invitedByUserId=${invite.invitedByUserId}`
        }
        return null
      },
      async save() {
        this.isLoading = true;
        try {
          await this.$api.service('user-invites').create({})
          await this.fetch()

          this.$snackbar.open({
            message: this.$t('auth.settings.saveSettingsSuccess'),
            type: "is-success"
          });
        } catch (err) {
          this.animate('shake')
          this.$toast.open({
            message: err.message,
            type: "is-danger"
          });
        }
        this.isLoading = false;
      }
    },
    computed: {
      ...mapGetters({
        user: "auth/user",
        settings: 'settings/get',
        showInvites: 'settings/showInvites'
      }),
      hasInvitesLeft () {
        return this.settings.invites.userCanInvite &&  this.invites.total < this.settings.invites.maxInvitesByUser
      }
    }
  };
</script>

<style lang="scss" scoped>
  @import "assets/styles/_animations";

  .ticket-wrapper {
    margin-left: -1.5rem;
    margin-right: -1.5rem;
    margin-bottom: -2rem;
    padding: 1.5rem;
    background-color: #eee;
  }
</style>
