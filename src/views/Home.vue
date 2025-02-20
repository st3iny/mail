<template>
	<Content app-name="mail">
		<Navigation />
		<Outbox v-if="$route.name === 'outbox'" />
		<MailboxThread v-else-if="activeAccount"
			:account="activeAccount"
			:mailbox="activeMailbox" />

		<template v-if="hasComposerSession">
			<ComposerSessionIndicator @close="onCloseMessageModal" />
			<NewMessageModal ref="newMessageModal" :accounts="accounts" />
		</template>
	</Content>
</template>

<script>
import { NcContent as Content } from '@nextcloud/vue'
import isMobile from '@nextcloud/vue/dist/Mixins/isMobile'

import '../../css/mail.scss'
import '../../css/mobile.scss'

import { testAccountConnection } from '../service/AccountService'
import logger from '../logger'
import MailboxThread from '../components/MailboxThread'
import Navigation from '../components/Navigation'
import Outbox from '../components/Outbox'
import ComposerSessionIndicator from '../components/ComposerSessionIndicator'
import { mapGetters } from 'vuex'

export default {
	name: 'Home',
	components: {
		Content,
		MailboxThread,
		Navigation,
		NewMessageModal: () => import(/* webpackChunkName: "new-message-modal" */ '../components/NewMessageModal.vue'),
		Outbox,
		ComposerSessionIndicator,
	},
	mixins: [isMobile],
	data() {
		return {
			hasComposerSession: false,
			accounts: null,
		}
	},
	computed: {
		...mapGetters(['composerSessionId']),
		activeAccount() {
			return this.$store.getters.getAccount(this.activeMailbox?.accountId)
		},
		activeMailbox() {
			return this.$store.getters.getMailbox(this.$route.params.mailboxId)
		},
		menu() {
			return this.buildMenu()
		},
	},
	watch: {
		async composerSessionId(id) {
			// Session was closed or discarded
			if (!id) {
				this.hasComposerSession = false
				return
			}

			// A new session is replacing the old session.  Fully reset the NewMessageModal
			// component in this case and wait for the template to fully render before showing the
			// modal again.
			if (this.hasComposerSession) {
				this.hasComposerSession = false
				await this.$nextTick()
			}

			this.hasComposerSession = true
		},
	},
	async beforeMount() {
		const accounts = this.$store.getters.accounts.filter((a) => !a.isUnified)
		this.accounts = await Promise.all(
			 accounts.map(async (account) => {
				return { ...account, connectionStatus: await testAccountConnection(account.id) }
			}))

	},
	created() {
		const accounts = this.$store.getters.accounts
		let startMailboxId = this.$store.getters.getPreference('start-mailbox-id')
		if (startMailboxId && !this.$store.getters.getMailbox(startMailboxId)) {
			// The start ID is set but the mailbox doesn't exist anymore
			startMailboxId = null
		}

		if (this.$route.name === 'home' && accounts.length > 1 && startMailboxId) {
			logger.debug('Loading start mailbox', { id: startMailboxId })
			this.$router.replace({
				name: 'mailbox',
				params: {
					mailboxId: startMailboxId,
				},
			})
		} else if (this.$route.name === 'home' && accounts.length > 1) {
			// Show first account
			const firstAccount = accounts[0]
			// FIXME: this assumes that there's at least one mailbox
			const firstMailbox = this.$store.getters.getMailboxes(firstAccount.id)[0]

			console.debug('loading first mailbox of first account', firstAccount.id, firstMailbox.databaseId)

			this.$router.replace({
				name: 'mailbox',
				params: {
					mailboxId: firstMailbox.databaseId,
				},
			})
		} else if (this.$route.name === 'home' && accounts.length === 1) {
			logger.debug('the only account we have is the unified one -> show the setup page')
			this.$router.replace({
				name: 'setup',
			})
		} else if (this.$route.name === 'mailto') {
			if (accounts.length === 0) {
				console.error('cannot handle mailto:, no accounts configured')
				return
			}

			// Show first account
			const firstAccount = accounts[0]
			// FIXME: this assumes that there's at least one mailbox
			const firstMailbox = this.$store.getters.getMailboxes(firstAccount.id)[0]

			console.debug('loading composer with first account and mailbox', firstAccount.id, firstMailbox.id)

			this.$router.replace({
				name: 'message',
				params: {
					mailboxId: firstMailbox.databaseId,
					threadId: 'mailto',
				},
				query: {
					to: this.$route.query.to,
					cc: this.$route.query.cc,
					bcc: this.$route.query.bcc,
					subject: this.$route.query.subject,
					body: this.$route.query.body,
				},
			})
		}
	},
	methods: {
		hideMessage() {
			this.$router.replace({
				name: 'mailbox',
				params: {
					mailboxId: this.$route.params.mailboxId,
					filter: this.$route.params?.filter,
				},
			})
		},
		async onCloseMessageModal() {
			await this.$refs.newMessageModal.onClose()
		},
	},
}

</script>

<style lang="scss" scoped>
:deep(.app-content-details) {
	margin: 0 auto;
	max-width: 900px;
	display: flex;
	flex-direction: column;
	flex: 1 1 100%;
	min-width: 70%;
}
// Align the appNavigation toggle with the apps header toolbar
:deep(button.app-navigation-toggle) {
	top: 8px;
}
</style>
