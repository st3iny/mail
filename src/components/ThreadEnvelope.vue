<!--
  - @copyright 2020 Christoph Wurst <christoph@winzerhof-wurst.at>
  -
  - @author 2020 Christoph Wurst <christoph@winzerhof-wurst.at>
  - @author 2021 Richard Steinmetz <richard@steinmetz.cloud>
  - @author 2022 Jonas Sulzer <jonas@violoncello.ch>
  -
  - @license AGPL-3.0-or-later
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <http://www.gnu.org/licenses/>.
  -->

<template>
	<div ref="envelope"
		class="envelope"
		:class="{'envelope--expanded' : expanded }">
		<div class="envelope__header">
			<div class="envelope__header__avatar">
				<Avatar v-if="envelope.from && envelope.from[0]"
					:email="envelope.from[0].email"
					:display-name="envelope.from[0].label"
					:disable-tooltip="true"
					:size="40" />
				<div
					v-if="isImportant"
					class="app-content-list-item-star icon-important"
					:data-starred="isImportant ? 'true' : 'false'"
					@click.prevent="hasWriteAcl ? onToggleImportant() : false"
					v-html="importantSvg" />
				<IconFavorite
					v-if="envelope.flags.flagged"
					fill-color="#f9cf3d"
					:size="18"
					class="app-content-list-item-star favorite-icon-style"
					:data-starred="envelope.flags.flagged ? 'true' : 'false'"
					@click.prevent="hasWriteAcl ? onToggleFlagged() : false" />
				<JunkIcon
					v-if="envelope.flags.$junk"
					:size="18"
					class="app-content-list-item-star junk-icon-style"
					:data-starred="envelope.flags.$junk ? 'true' : 'false'"
					@click.prevent="hasWriteAcl ? onToggleJunk() : false" />
			</div>

			<router-link
				:to="route"
				event=""
				class="left"
				:class="{seen: envelope.flags.seen}"
				@click.native.prevent="$emit('toggle-expand', $event)">
				<div class="envelope__header__left__sender-subject-tags">
					<div class="sender">
						{{ envelope.from && envelope.from[0] ? envelope.from[0].label : '' }}
						<p class="sender__email">
							{{ envelope.from && envelope.from[0] ? envelope.from[0].email : '' }}
						</p>
					</div>
					<div v-if="hasChangedSubject" class="subline">
						{{ cleanSubject }}
					</div>
					<div v-if="showSubline" class="subline">
						<span class="preview">
							{{ isEncrypted ? t('mail', 'Encrypted message') : envelope.previewText }}
						</span>
					</div>
					<div v-for="tag in tags"
						:key="tag.id"
						class="tag-group">
						<div class="tag-group__bg"
							:style="{'background-color': tag.color}" />
						<span class="tag-group__label"
							:style="{color: tag.color}">{{ tag.displayName }} </span>
					</div>
				</div>
				<div class="envelope__header__left__unsubscribe">
					<NcButton
						v-if="message && message.dkimValid && (message.unsubscribeUrl || message.unsubscribeMailto)"
						type="tertiary"
						class="envelope__header__unsubscribe"
						@click="showListUnsubscribeConfirmation = true">
						{{ t('mail', 'Unsubscribe') }}
					</NcButton>
				</div>
			</router-link>
			<div class="right">
				<Moment class="timestamp" :timestamp="envelope.dateInt" />
				<template v-if="expanded">
					<NcActions v-if="smimeData.isSigned || smimeData.isEncrypted">
						<template #icon>
							<LockPlusIcon v-if="smimeData.isEncrypted"
								:size="20"
								fill-color="#008000" />
							<LockIcon v-else-if="smimeData.signatureIsValid"
								:size="20"
								fill-color="#008000" />
							<LockOffIcon v-else
								:size="20"
								fill-color="red" />
						</template>
						<NcActionText class="smime-text" :title="smimeHeading">
							{{ smimeMessage }}
						</NcActionText>
						<!-- TODO: display information about signer and/or CA certificate -->
					</NcActions>
					<NcActions :inline="inlineMenuSize">
						<NcActionButton v-if="inlineMenuSize >= 1 || !moreActionsOpen"
							:class="{ primary: expanded}"
							:aria-label="hasMultipleRecipients ? t('mail', 'Reply all') : t('mail', 'Reply')"
							:title="hasMultipleRecipients ? t('mail', 'Reply all') : t('mail', 'Reply')"
							type="tertiary-no-background"
							@click="onReply">
							<template #icon>
								<ReplyAllIcon v-if="hasMultipleRecipients"
									:size="20" />
								<ReplyIcon v-else
									:size="20" />
							</template>
						</NcActionButton>
						<NcActionButton v-if="hasWriteAcl && (inlineMenuSize >= 2 || !moreActionsOpen)"
							type="tertiary-no-background"
							class="action--primary"
							:aria-label="envelope.flags.flagged ? t('mail', 'Mark as unfavorite') : t('mail', 'Mark as favorite')"
							:title="envelope.flags.flagged ? t('mail', 'Mark as unfavorite') : t('mail', 'Mark as favorite')"
							:close-after-click="true"
							@click.prevent="onToggleFlagged">
							<template #icon>
								<StarOutline v-if="showFavoriteIconVariant"
									:size="20" />
								<IconFavorite v-else
									:size="20" />
							</template>
						</NcActionButton>
						<NcActionButton v-if="hasSeenAcl && (inlineMenuSize >= 3 || !moreActionsOpen)"
							type="tertiary-no-background"
							class="action--primary"
							:aria-label="envelope.flags.seen ? t('mail', 'Mark as unread') : t('mail', 'Mark as read')"
							:title="envelope.flags.seen ? t('mail', 'Mark as unread') : t('mail', 'Mark as read')"
							:close-after-click="true"
							@click.prevent="onToggleSeen">
							<template #icon>
								<EmailRead v-if="showImportantIconVariant"
									:size="20" />
								<EmailUnread v-else
									:size="20" />
							</template>
						</NcActionButton>
						<NcActionButton v-if="showArchiveButton && hasArchiveAcl && (inlineMenuSize >= 4 || !moreActionsOpen)"
							:close-after-click="true"
							:title="t('mail', 'Archive message')"
							:disabled="disableArchiveButton"
							:aria-label="t('mail', 'Archive message')"
							type="tertiary-no-background"
							@click.prevent="onArchive">
							<template #icon>
								<ArchiveIcon
									:title="t('mail', 'Archive message')"
									:size="20" />
							</template>
						</NcActionButton>
						<NcActionButton v-if="hasDeleteAcl && (inlineMenuSize >= 5 || !moreActionsOpen)"
							:close-after-click="true"
							:title="t('mail', 'Delete message')"
							:aria-label="t('mail', 'Delete message')"
							type="tertiary-no-background"
							@click.prevent="onDelete">
							<template #icon>
								<DeleteIcon
									:title="t('mail', 'Delete message')"
									:size="20" />
							</template>
						</NcActionButton>
						<MenuEnvelope class="app-content-list-item-menu"
							:envelope="envelope"
							:mailbox="mailbox"
							:with-reply="false"
							:with-select="false"
							:with-show-source="true"
							:more-actions-open.sync="moreActionsOpen"
							@delete="$emit('delete',envelope.databaseId)"
							@show-source-modal="onShowSourceModal"
							@open-tag-modal="onOpenTagModal"
							@open-move-modal="onOpenMoveModal"
							@open-event-modal="onOpenEventModal"
							@open-task-modal="onOpenTaskModal" />
					</NcActions>
					<NcModal v-if="showSourceModal" class="source-modal" @close="onCloseSourceModal">
						<div class="source-modal-content">
							<div class="section">
								<h2>{{ t('mail', 'Message source') }}</h2>
								<pre class="message-source">{{ rawMessage }}</pre>
							</div>
						</div>
					</NcModal>
					<MoveModal v-if="showMoveModal"
						:account="account"
						:envelopes="[envelope]"
						@move="onMove"
						@close="onCloseMoveModal" />
					<EventModal v-if="showEventModal"
						:envelope="envelope"
						@close="onCloseEventModal" />
					<TaskModal v-if="showTaskModal"
						:envelope="envelope"
						@close="onCloseTaskModal" />
					<TagModal
						v-if="showTagModal"
						:account="account"
						:envelopes="[envelope]"
						@close="onCloseTagModal" />
				</template>
			</div>
		</div>
		<MessageLoadingSkeleton v-if="loading !== LOADING_DONE" />
		<Message v-if="message && loading !== LOADING_MESSAGE"
			v-show="loading === LOADING_DONE"
			:envelope="envelope"
			:message="message"
			:full-height="fullHeight"
			@load="loading = LOADING_DONE" />
		<Error v-else-if="error"
			:error="error.message || t('mail', 'Not found')"
			message=""
			:data="error"
			:auto-margin="true"
			role="alert" />
		<ConfirmModal v-if="message && message.unsubscribeUrl && message.isOneClickUnsubscribe && showListUnsubscribeConfirmation"
			:confirm-text="t('mail', 'Unsubscribe')"
			:title="t('mail', 'Unsubscribe via link')"
			@cancel="showListUnsubscribeConfirmation = false"
			@confirm="unsubscribeViaOneClick">
			{{ t('mail', 'Unsubscribing will stop all messages from the mailing list {sender}', { sender: from }) }}
		</ConfirmModal>
		<ConfirmModal v-else-if="message && message.unsubscribeUrl && showListUnsubscribeConfirmation"
			:confirm-text="t('mail', 'Unsubscribe')"
			:confirm-url="message.unsubscribeUrl"
			:title="t('mail', 'Unsubscribe via link')"
			@cancel="showListUnsubscribeConfirmation = false"
			@confirm="showListUnsubscribeConfirmation = false">
			{{ t('mail', 'Unsubscribing will stop all messages from the mailing list {sender}', { sender: from }) }}
		</ConfirmModal>
		<ConfirmModal v-else-if="message && message.unsubscribeMailto && showListUnsubscribeConfirmation"
			:confirm-text="t('mail', 'Send unsubscribe email')"
			:title="t('mail', 'Unsubscribe via email')"
			:disabled="unsubscribing"
			@cancel="showListUnsubscribeConfirmation = false"
			@confirm="unsubscribeViaMailto">
			{{ t('mail', 'Unsubscribing will stop all messages from the mailing list {sender}', { sender: from }) }}
		</ConfirmModal>
	</div>
</template>
<script>
import Avatar from './Avatar'
import { NcActionButton, NcButton, NcModal } from '@nextcloud/vue'
import ConfirmModal from './ConfirmationModal.vue'
import Error from './Error'
import importantSvg from '../../img/important.svg'
import IconFavorite from 'vue-material-design-icons/Star'
import JunkIcon from './icons/JunkIcon'
import MessageLoadingSkeleton from './MessageLoadingSkeleton'
import logger from '../logger'
import Message from './Message'
import MenuEnvelope from './MenuEnvelope'
import Moment from './Moment'
import { mailboxHasRights } from '../util/acl'
import ReplyIcon from 'vue-material-design-icons/Reply'
import ReplyAllIcon from 'vue-material-design-icons/ReplyAll'
import StarOutline from 'vue-material-design-icons/StarOutline'
import DeleteIcon from 'vue-material-design-icons/Delete'
import ArchiveIcon from 'vue-material-design-icons/PackageDown'
import EmailUnread from 'vue-material-design-icons/Email'
import EmailRead from 'vue-material-design-icons/EmailOpen'
import LockIcon from 'vue-material-design-icons/Lock'
import LockPlusIcon from 'vue-material-design-icons/LockPlus'
import LockOffIcon from 'vue-material-design-icons/LockOff'
import { buildRecipients as buildReplyRecipients } from '../ReplyBuilder'
import { hiddenTags } from './tags.js'
import { showError, showSuccess } from '@nextcloud/dialogs'
import { matchError } from '../errors/match'
import NoTrashMailboxConfiguredError from '../errors/NoTrashMailboxConfiguredError'
import { isPgpText } from '../crypto/pgp'
import NcActions from '@nextcloud/vue/dist/Components/NcActions'
import NcActionText from '@nextcloud/vue/dist/Components/NcActionText'
import { unsubscribe } from '../service/ListService'
import TagModal from './TagModal.vue'
import MoveModal from './MoveModal.vue'
import TaskModal from './TaskModal.vue'
import EventModal from './EventModal.vue'
import axios from '@nextcloud/axios'
import { generateUrl } from '@nextcloud/router'

// Ternary loading state
const LOADING_DONE = 0
const LOADING_MESSAGE = 1
const LOADING_BODY = 2

export default {
	name: 'ThreadEnvelope',
	components: {
		NcModal,
		EventModal,
		TaskModal,
		MoveModal,
		TagModal,
		ConfirmModal,
		Avatar,
		NcActionButton,
		NcButton,
		Error,
		IconFavorite,
		JunkIcon,
		MessageLoadingSkeleton,
		MenuEnvelope,
		Moment,
		Message,
		ReplyIcon,
		ReplyAllIcon,
		StarOutline,
		EmailRead,
		EmailUnread,
		DeleteIcon,
		ArchiveIcon,
		LockIcon,
		LockOffIcon,
		LockPlusIcon,
		NcActions,
		NcActionText,
	},
	props: {
		envelope: {
			required: true,
			type: Object,
		},
		mailboxId: {
			required: false,
			type: [
				String,
				Number,
			],
			default: undefined,
		},
		expanded: {
			required: false,
			type: Boolean,
			default: false,
		},
		fullHeight: {
			required: false,
			type: Boolean,
			default: false,
		},
		withSelect: {
			// "Select" action should only appear in envelopes from the envelope list
			type: Boolean,
			default: true,
		},
		threadSubject: {
			required: true,
			type: String,
		},
	},
	data() {
		return {
			loading: LOADING_DONE,
			showListUnsubscribeConfirmation: false,
			error: undefined,
			message: undefined,
			importantSvg,
			unsubscribing: false,
			seenTimer: undefined,
			LOADING_BODY,
			LOADING_DONE,
			LOADING_MESSAGE,
			recomputeMenuSize: 0,
			moreActionsOpen: false,
			showSourceModal: false,
			showMoveModal: false,
			showEventModal: false,
			showTaskModal: false,
			showTagModal: false,
			rawMessage: '', // Will hold the raw source of the message when requested
		}
	},
	computed: {
		inlineMenuSize() {
			// eslint-disable-next-line no-unused-expressions
			const { envelope } = this.$refs
			const envelopeWidth = (envelope && envelope.clientWidth) || 250
			const spaceToFill = envelopeWidth - 500 + this.recomputeMenuSize
			return Math.floor(spaceToFill / 44)
		},
		account() {
			return this.$store.getters.getAccount(this.envelope.accountId)
		},
		from() {
			if (!this.message || !this.message.from.length) {
				return '?'
			}
			if (this.message.from[0].label) {
				return this.message.from[0].label
			}
			return this.message.from[0].email
		},
		hasMultipleRecipients() {
			if (!this.account) {
				console.error('account is undefined', {
					accountId: this.envelope.accountId,
				})
			}
			const recipients = buildReplyRecipients(this.envelope, {
				label: this.account.name,
				email: this.account.emailAddress,
			})
			return recipients.to.concat(recipients.cc).length > 1
		},
		route() {
			return {
				name: 'message',
				params: {
					mailboxId: this.mailboxId || this.envelope.mailboxId,
					threadId: this.envelope.databaseId,
				},
			}
		},
		isEncrypted() {
			return this.envelope.previewText
				&& isPgpText(this.envelope.previewText)
		},
		isImportant() {
			return this.$store.getters
				.getEnvelopeTags(this.envelope.databaseId)
				.find((tag) => tag.imapLabel === '$label1')
		},
		tags() {
			return this.$store.getters.getEnvelopeTags(this.envelope.databaseId).filter(
				(tag) => tag.imapLabel !== '$label1' && !(tag.displayName.toLowerCase() in hiddenTags)
			)
		},
		hasChangedSubject() {
			return this.cleanSubject !== this.cleanThreadSubject
		},
		cleanSubject() {
			return this.filterSubject(this.envelope.subject)
		},
		cleanThreadSubject() {
			return this.filterSubject(this.threadSubject)
		},
		showSubline() {
			return !this.expanded && !!this.envelope.previewText
		},
		showArchiveButton() {
			return this.account.archiveMailboxId !== null
		},
		disableArchiveButton() {
			return this.account.archiveMailboxId !== null
				&& this.account.archiveMailboxId === this.mailbox.databaseId
		},
		junkFavoritePosition() {
			return this.showSubline && this.tags.length > 0
		},
		showFavoriteIconVariant() {
			return this.envelope.flags.flagged
		},
		showImportantIconVariant() {
			return this.envelope.flags.seen
		},
		hasSeenAcl() {
			return mailboxHasRights(this.mailbox, 's')
		},
		hasArchiveAcl() {
			const hasDeleteSourceAcl = () => {
				return mailboxHasRights(this.mailbox, 'te')
			}

			const hasCreateDestinationAcl = () => {
				return mailboxHasRights(this.archiveMailbox, 'i')
			}

			return hasDeleteSourceAcl() && hasCreateDestinationAcl()
		},
		hasDeleteAcl() {
			return mailboxHasRights(this.mailbox, 'te')
		},
		hasWriteAcl() {
			return mailboxHasRights(this.mailbox, 'w')
		},
		mailbox() {
			return this.$store.getters.getMailbox(this.mailboxId)
		},
		archiveMailbox() {
			return this.$store.getters.getMailbox(this.account.archiveMailboxId)
		},
		/**
		 * @return {{isSigned: (boolean|undefined), signatureIsValid: (boolean|undefined)}}
		 */
		smimeData() {
			return this.message?.smime ?? {}
		},
		smimeHeading() {
			if (this.smimeData.isEncrypted) {
				return t('mail', 'Encrypted & verified ')
			}

			if (this.smimeData.signatureIsValid) {
				return t('mail', 'Signature verified')
			}

			return t('mail', 'Signature unverified ')
		},
		smimeMessage() {
			if (this.smimeData.isEncrypted) {
				return t('mail', 'This message was encrypted by the sender before it was sent.')
			}

			if (this.smimeData.signatureIsValid) {
				return t('mail', 'This message contains a verified digital S/MIME signature. The message wasn\'t changed since it was sent.')
			}

			return t('mail', 'This message contains an unverified digital S/MIME signature. The message might have been changed since it was sent or the certificate of the signer is untrusted.')
		},
	},
	watch: {
		expanded(expanded) {
			if (expanded) {
				this.fetchMessage()
			} else {
				this.message = undefined
				this.loading = LOADING_DONE
			}
		},
	},
	async mounted() {
		window.addEventListener('resize', this.redrawMenuBar)
		if (this.expanded) {
			await this.fetchMessage()

			// Only one envelope is expanded at the time of mounting so we can
			// assume that this is the relevant envelope to be scrolled to.
			this.$nextTick(() => this.scrollToCurrentEnvelope())
		}
		this.$checkInterval = setInterval(() => {
			const { envelope } = this.$refs
			const isWidthAvailable = (envelope && envelope.clientWidth > 0)
			if (isWidthAvailable) {
				this.redrawMenuBar()
				clearInterval(this.$checkInterval)
			}
		}, 100)
	},
	beforeDestroy() {
		if (this.seenTimer !== undefined) {
			logger.info('Navigating away before seenTimer delay, will not mark message as seen/read')
			clearTimeout(this.seenTimer)
		}
		window.removeEventListener('resize', this.redrawMenuBar)
	},
	methods: {
		redrawMenuBar() {
			this.$nextTick(() => {
				this.recomputeMenuSize++
			})
		},
		filterSubject(value) {
			return value.replace(/((?:[\t ]*(?:R|RE|F|FW|FWD):[\t ]*)*)/i, '')
		},
		async fetchMessage() {
			this.loading = LOADING_MESSAGE
			this.error = undefined

			logger.debug(`fetching thread message ${this.envelope.databaseId}`)

			try {
				this.message = await this.$store.dispatch('fetchMessage', this.envelope.databaseId)
				logger.debug(`message ${this.envelope.databaseId} fetched`, { message: this.message })

				if (!this.envelope.flags.seen && this.hasSeenAcl) {
					logger.info('Starting timer to mark message as seen/read')
					this.seenTimer = setTimeout(() => {
						this.$store.dispatch('toggleEnvelopeSeen', { envelope: this.envelope })
						this.seenTimer = undefined
					}, 2000)
				}

				if (this.message.hasHtmlBody) {
					this.loading = LOADING_BODY
				} else {
					this.loading = LOADING_DONE
				}
			} catch (error) {
				this.error = error
				this.loading = LOADING_DONE
				logger.error('Could not fetch message', { error })
			}

			// Fetch itineraries if they haven't been included in the message data
			if (this.message && !this.message.itineraries) {
				this.fetchItineraries()
			}
			// Fetch dkim
			if (this.message && this.message.dkimValid === undefined) {
				this.fetchDkim()
			}
		},
		async fetchItineraries() {
			// Sanity check before actually making the request
			if (!this.message.hasHtmlBody && this.message.attachments.length === 0) {
				return
			}

			logger.debug(`Fetching itineraries for message ${this.envelope.databaseId}`)

			try {
				const itineraries = await this.$store.dispatch('fetchItineraries', this.envelope.databaseId)
				logger.debug(`Itineraries of message ${this.envelope.databaseId} fetched`, { itineraries })
			} catch (error) {
				logger.error(`Could not fetch itineraries of message ${this.envelope.databaseId}`, { error })
			}
		},
		async fetchDkim() {
			if (this.message.hasDkimSignature === false) {
				return
			}

			logger.debug(`Fetching DKIM for message ${this.envelope.databaseId}`)

			try {
				const dkim = await this.$store.dispatch('fetchDkim', this.envelope.databaseId)
				logger.debug(`DKIM of message ${this.envelope.databaseId} fetched`, { dkim })
			} catch (error) {
				logger.error(`Could not fetch DKIM of message ${this.envelope.databaseId}`, { error })
			}
		},
		scrollToCurrentEnvelope() {
			// Account for global navigation bar and thread header
			const globalHeader = document.querySelector('#header').clientHeight
			const threadHeader = document.querySelector('#mail-thread-header').clientHeight
			const top = this.$el.getBoundingClientRect().top - globalHeader - threadHeader
			window.scrollTo({ top })
		},
		onReply() {
			this.$store.dispatch('startComposerSession', {
				reply: {
					mode: this.hasMultipleRecipients ? 'replyAll' : 'reply',
					data: this.envelope,
				},
			})
		},
		onToggleImportant() {
			this.$store.dispatch('toggleEnvelopeImportant', this.envelope)
		},
		onToggleFlagged() {
			this.$store.dispatch('toggleEnvelopeFlagged', this.envelope)
		},
		onToggleJunk() {
			this.$store.dispatch('toggleEnvelopeJunk', this.envelope)
		},
		onToggleSeen() {
			this.$store.dispatch('toggleEnvelopeSeen', { envelope: this.envelope })
		},
		async onDelete() {
			// Remove from selection first
			if (this.withSelect) {
				this.$emit('unselect')
			}

			// Delete
			this.$emit('delete', this.envelope.databaseId)

			logger.info(`deleting message ${this.envelope.databaseId}`)

			try {
				await this.$store.dispatch('deleteMessage', {
					id: this.envelope.databaseId,
				})
			} catch (error) {
				showError(await matchError(error, {
					[NoTrashMailboxConfiguredError.getName()]() {
						return t('mail', 'No trash mailbox configured')
					},
					default(error) {
						logger.error('could not delete message', error)
						return t('mail', 'Could not delete message')
					},
				}))
			}
		},
		async onArchive() {
			// Remove from selection first
			if (this.withSelect) {
				this.$emit('unselect')
			}

			// Archive
			this.$emit('archive', this.envelope.databaseId)

			logger.info(`archiving message ${this.envelope.databaseId}`)

			try {
				await this.$store.dispatch('moveMessage', {
					id: this.envelope.databaseId,
					destMailboxId: this.account.archiveMailboxId,
				})
			} catch (error) {
				logger.error('could not archive message', error)
				return t('mail', 'Could not archive message')
			}
		},
		async unsubscribeViaOneClick() {
			try {
				this.unsubscribing = true

				await unsubscribe(this.envelope.databaseId)
				showSuccess(t('mail', 'Unsubscribe request sent'))
			} catch (error) {
				logger.error('Could not one-click unsubscribe', { error })
				showError(t('mail', 'Could not unsubscribe from mailing list'))
			} finally {
				this.unsubscribing = false
				this.showListUnsubscribeConfirmation = false
			}
		},
		async unsubscribeViaMailto() {
			const mailto = this.message.unsubscribeMailto
			const [email, paramString] = mailto.replace(/^mailto:/, '').split('?')
			let params = {}
			const now = new Date().getTime() / 1000
			if (paramString) {
				params = paramString.split('&').map(encoded => ({
					key: encoded.split('=')[0].toLowerCase(),
					value: decodeURIComponent(encoded.split('=')[1]),
				}))
			}
			try {
				this.unsubscribing = true
				const message = await this.$store.dispatch('outbox/enqueueMessage', {
					message: {
						accountId: this.message.accountId,
						subject: params.subject || 'Unsubscribe',
						body: params.body || '',
						editorBody: params.body || '',
						isHtml: false,
						to: [{
							label: email,
							email,
						}],
						cc: [],
						bcc: [],
						attachments: [],
						aliasId: null,
						inReplyToMessageId: null,
						sendAt: now,
						draftId: null,
						smimeEncrypt: false,
						smimeSign: false,
					},
				})
				logger.debug('Unsubscribe email to ' + email + ' enqueued')
				await this.$store.dispatch('outbox/sendMessage', { id: message.id })
				logger.debug('Unsubscribe email sent to ' + email)
				showSuccess(t('mail', 'Unsubscribe request sent'))
			} catch (error) {
				logger.error('Could not enqueue or send unsubscribe email', { error })
				showError(t('mail', 'Could not unsubscribe from mailing list'))
			} finally {
				this.unsubscribing = false
				this.showListUnsubscribeConfirmation = false
			}
		},
		onMove() {
			this.$emit('move')
		},
		onOpenMoveModal() {
			this.showMoveModal = true
		},
		onCloseMoveModal() {
			this.showMoveModal = false
		},
		onOpenEventModal() {
			this.showEventModal = true
		},
		onCloseEventModal() {
			this.showEventModal = false
		},
		onOpenTaskModal() {
			this.showTaskModal = true
		},
		onCloseTaskModal() {
			this.showTaskModal = false
		},
		onOpenTagModal() {
			this.showTagModal = true
		},
		onCloseTagModal() {
			this.showTagModal = false
		},
		async onShowSourceModal() {
			if (this.rawMessage.length === 0) {
				const resp = await axios.get(
					generateUrl('/apps/mail/api/messages/{id}/source', {
						id: this.envelope.databaseId,
					})
				)
				this.rawMessage = resp.data.source
			}
			this.showSourceModal = true
		},
		onCloseSourceModal() {
			this.showSourceModal = false
		},
	},
}
</script>

<style lang="scss" scoped>
	.sender {
		margin-left: 8px;

		&__email{
			color: var(--color-text-maxcontrast);
		}
	}

	.right {
		display: flex;
		flex-direction: row;
		align-items: center;
		justify-content: flex-end;
		margin-left: 10px;
		height: 44px;

		.app-content-list-item-menu {
			margin-left: 4px;
		}

		.timestamp {
			margin-right: 10px;
			color: var(--color-text-maxcontrast);
			white-space: nowrap;
			margin-bottom: 0;
		}
	}
	.button {
		color: var(--color-main-background);
		&:not(.active):not(.primary) {
			display: none;

			&.primary {
				background-color: var(--color-primary-element);
				opacity: 1;
				margin-bottom: 0;

			}
		}
	}

	.envelope {
		display: flex;
		flex-direction: column;
		border: 2px solid var(--color-border);
		border-radius: 16px;
		margin-left: 10px;
		margin-right: 10px;
		background-color: var(--color-main-background);
		padding-bottom: 28px;
		animation: show 200ms 90ms cubic-bezier(.17, .67, .83, .67) forwards;
		opacity: 0.5;
		transform-origin: top center;
		@keyframes show {
			100% {
				opacity: 1;
				transform: none;
			}
		}

		& + .envelope {
			margin-top: -28px;
		}

		&:last-of-type {
			margin-bottom: 10px;
			padding-bottom: 0;
		}

		&__header {
			position: relative;
			display: flex;
			align-items: center;
			padding: 10px;
			border-radius: var(--border-radius);
			min-height: 68px; /* prevents jumping between open/collapsed */

			&__avatar {
				position: relative;

				.app-content-list-item-star {
					position: absolute;
					cursor: pointer;

					&.icon-important {
						background-image: none;
						opacity: 1;
						width: 16px;
						height: 16px;
						display: flex;
						top: 0px;
						left: 0px;

						&:hover,
						&:focus {
							opacity: 0.5;
						}

						:deep(path) {
							fill: #ffcc00;
							stroke: var(--color-main-background);
							cursor: pointer;
						}
					}
					&.favorite-icon-style {
						display: inline-block;
						top: -2px;
						right: -2px;

						stroke: var(--color-main-background);
						stroke-width: 2;
						&:hover {
							opacity: .5;
						}
					}
					&.junk-icon-style {
						display: inline-block;
						bottom: -2px;
						right: -2px;
						opacity: .2;
						&:hover {
							opacity: .1;
						}
					}
				}
			}

			&__unsubscribe {
				color: var(--color-text-maxcontrast);
			}
			&__left__sender-subject-tags {
				text-overflow: ellipsis;
				overflow: hidden;
				white-space: nowrap;
				width: 100%;
			}
		}

		.subline {
			margin-left: 8px;
			color: var(--color-text-maxcontrast);
			cursor: default;
			overflow: hidden;
			text-overflow: ellipsis;
			white-space: nowrap;
		}

		&--expanded {
			min-height: 350px;
		}
	}
	.left {
		flex-grow: 1;
		min-width: 0; /* https://css-tricks.com/flexbox-truncated-text/ */
		display: flex;
		position: relative;
		z-index: 1;
		padding: 2em;
		margin: -2em;
		margin-right: 0;
		align-items: center;
	}
	.left:not(.seen) {
		font-weight: bold;
	}
	.tag-group__label {
		margin: 0 7px;
		z-index: 2;
		font-size: calc(var(--default-font-size) * 0.8);
		font-weight: bold;
		padding-left: 2px;
		padding-right: 2px;
	}
	.tag-group__bg {
		position: absolute;
		width: 100%;
		height: 100%;
		top: 0;
		left: 0;
		opacity: 15%;
	}
	.tag-group {
		display: inline-block;
		border: 1px solid transparent;
		border-radius: var(--border-radius-pill);
		position: relative;
		margin: 0 1px;
		overflow: hidden;
		left: 4px;
	}
	.smime-text {
		// same as padding-right on action-text styling
		padding-left: 14px;
	}
	:deep(.action-button__title) {
		font-weight: normal;
		display: inline;
		align-items: center;
	}
</style>
