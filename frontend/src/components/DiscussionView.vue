<template>
  <div class="relative flex h-full flex-col" v-if="postId && discussion">
    <div
      class="fixed top-0 z-10 -mx-4 w-full border-b bg-white sm:mx-0"
      v-show="showNavbar"
    >
      <transition
        enter-active-class="transition ease-out duration-200"
        enter-from-class="opacity-0 translate-y-4"
        enter-to-class="opacity-100 translate-y-0"
      >
        <div class="flex items-center px-6 py-4 sm:px-2" v-show="showNavbar">
          <UserProfileLink :user="discussion.owner">
            <UserAvatar :user="discussion.owner" />
          </UserProfileLink>
          <h1 class="ml-3 flex items-center text-2xl font-bold">
            <Tooltip
              v-if="discussion.closed_at"
              text="This discussion is closed"
            >
              <FeatherIcon
                name="lock"
                class="mr-2 h-4 w-4 text-gray-700"
                :stroke-width="2"
              />
            </Tooltip>
            {{ discussion.title }}
          </h1>
        </div>
      </transition>
    </div>

    <div class="py-6">
      <div class="mb-3 flex items-center">
        <UserProfileLink class="mr-3" :user="discussion.owner">
          <UserAvatar :user="discussion.owner" />
        </UserProfileLink>
        <div class="flex w-full items-center">
          <div>
            <span class="text-base text-gray-900">
              <UserProfileLink
                class="font-medium hover:text-blue-600"
                :user="discussion.owner"
              >
                {{ $user(discussion.owner).full_name }}
              </UserProfileLink>
              in
              <router-link
                class="hover:text-blue-600"
                :to="{
                  name: 'Team',
                  params: {
                    teamId: discussion.team,
                  },
                }"
              >
                {{ $getDoc('Team', discussion.team)?.title || discussion.team }}
              </router-link>
              <span class="text-gray-500"> &mdash; </span>
              <router-link
                class="hover:text-blue-600"
                :to="{
                  name: 'ProjectOverview',
                  params: {
                    teamId: discussion.team,
                    projectId: discussion.project,
                  },
                }"
              >
                {{
                  $getDoc('Team Project', discussion.project)?.title ||
                  discussion.project
                }}
              </router-link>
            </span>
            &middot;
            <span
              class="text-base text-gray-600"
              :title="$dayjs(discussion.creation)"
            >
              {{ $dayjs(discussion.creation).fromNow() }}
            </span>
          </div>
          <div class="ml-auto flex space-x-2">
            <Dropdown
              v-if="!readOnlyMode"
              v-show="!editingContent && !editingTitle"
              class="ml-auto"
              placement="right"
              :button="{
                icon: 'more-horizontal',
                appearance: 'minimal',
                label: 'Discussion Options',
              }"
              :options="[
                {
                  label: 'Edit Title',
                  icon: 'edit',
                  handler: () => (editingTitle = true),
                },
                {
                  label: 'Edit Post',
                  icon: 'edit',
                  handler: () => (editingContent = true),
                  condition: () => $user().name === discussion.owner,
                },
                {
                  label: 'Copy link',
                  icon: 'link',
                  handler: () => copyLink(),
                },
                {
                  label: 'Close discussion',
                  icon: 'lock',
                  condition: () => !discussion.closed_at,
                  handler: () => {
                    $dialog({
                      title: 'Close discussion',
                      message:
                        'When a discussion is closed, commenting is disabled. Anyone can re-open the discussion later. Do you want to close this discussion?',
                      icon: { name: 'lock', appearance: 'success' },
                      actions: [
                        {
                          label: 'Close',
                          handler: ({ close }) =>
                            $resources.discussion.closeDiscussion
                              .submit()
                              .then(close),
                          appearance: 'primary',
                        },
                        {
                          label: 'Cancel',
                        },
                      ],
                    })
                  },
                },
                {
                  label: 'Re-open discussion',
                  icon: 'unlock',
                  condition: () => discussion.closed_at,
                  handler: () => {
                    $dialog({
                      title: 'Re-open discussion',
                      message:
                        'Do you want to re-open this discussion? Anyone can comment on it again.',
                      icon: { name: 'unlock', appearance: 'success' },
                      actions: [
                        {
                          label: 'Re-open',
                          handler: ({ close }) =>
                            $resources.discussion.reopenDiscussion
                              .submit()
                              .then(close),
                          appearance: 'primary',
                        },
                        {
                          label: 'Cancel',
                        },
                      ],
                    })
                  },
                },
                {
                  label: 'Move to another project',
                  icon: 'log-out',
                  handler: () => (discussionMoveDialog.show = true),
                },
              ]"
            />
            <Button
              v-if="editingContent || editingTitle"
              @click="
                () => {
                  $resources.discussion.reload()
                  editingContent = false
                  editingTitle = false
                }
              "
            >
              Discard
            </Button>
            <Button
              v-if="editingContent || editingTitle"
              appearance="primary"
              @click="
                () => {
                  let values = {}
                  if (editingContent) {
                    values.content = discussion.content
                  }
                  if (editingTitle) {
                    values.title = discussion.title
                  }
                  $resources.discussion.setValue.submit(values).then(() => {
                    this.updateUrlSlug()
                  })
                  editingContent = false
                  editingTitle = false
                }
              "
            >
              Save
            </Button>
          </div>
        </div>
      </div>
      <div>
        <div v-if="editingTitle" class="mb-3 w-full">
          <input
            type="text"
            class="mt-1 w-full rounded-lg border-0 bg-gray-100 px-2 py-1 text-xl font-semibold focus:ring-0"
            ref="title"
            v-model="discussion.title"
            placeholder="Title"
          />
        </div>
        <h1
          v-else
          v-visibility="handleTitleVisibility"
          class="flex items-center text-3xl font-bold"
        >
          <Tooltip v-if="discussion.closed_at" text="This discussion is closed">
            <FeatherIcon
              name="lock"
              class="mr-2 h-4 w-4 text-gray-700"
              :stroke-width="2"
            />
          </Tooltip>
          <span>
            {{ discussion.title }}
          </span>
        </h1>
      </div>
      <TextEditor
        :key="editingContent"
        :editor-class="[
          'min-h-[8rem] prose-sm',
          { 'border px-3 py-2 rounded-b-lg': editingContent },
        ]"
        :editable="editingContent"
        :content="discussion.content"
        @change="discussion.content = $event"
        :bubbleMenu="editingContent"
        :fixedMenu="editingContent"
      />
      <div class="mt-3">
        <Reactions
          doctype="Team Discussion"
          :name="discussion.name"
          v-model:reactions="discussion.reactions"
          :read-only-mode="readOnlyMode"
        />
      </div>
    </div>
    <CommentsArea
      doctype="Team Discussion"
      :name="discussion.name"
      :newCommentsFrom="discussion.last_unread_comment"
      :read-only-mode="readOnlyMode"
      :disable-new-comment="discussion.closed_at"
    />
    <Dialog
      :options="{
        title: 'Move discussion to another project',
      }"
      @close="
        () => {
          discussionMoveDialog.project = null
          $resources.discussion.moveToProject.reset()
        }
      "
      v-model="discussionMoveDialog.show"
    >
      <template #body-content>
        <Autocomplete
          :options="projectOptions"
          v-model="discussionMoveDialog.project"
          placeholder="Select a project"
        />
        <ErrorMessage
          class="mt-2"
          :message="$resources.discussion.moveToProject.error"
        />
      </template>
      <template #actions>
        <Button
          appearance="primary"
          :loading="$resources.discussion.moveToProject.loading"
          @click="
            $resources.discussion.moveToProject.submit({
              project: discussionMoveDialog.project?.value,
            })
          "
        >
          {{
            discussionMoveDialog.project
              ? `Move to ${discussionMoveDialog.project.label}`
              : 'Move'
          }}
        </Button>
        <Button @click="discussionMoveDialog.show = false">Cancel</Button>
      </template>
    </Dialog>
  </div>
</template>
<script>
import {
  Autocomplete,
  Dropdown,
  Dialog,
  Tooltip,
  visibilityDirective,
} from 'frappe-ui'
import Reactions from './Reactions.vue'
import CommentsArea from '@/components/CommentsArea.vue'
import CommentEditor from './CommentEditor.vue'
import TextEditor from '@/components/TextEditor.vue'
import UserProfileLink from './UserProfileLink.vue'
import { copyToClipboard } from '@/utils'
import { teams } from '@/data/teams'
import { getTeamProjects } from '@/data/projects'

export default {
  name: 'DiscussionView',
  props: ['postId', 'readOnlyMode'],
  directives: {
    visibility: visibilityDirective,
  },
  components: {
    TextEditor,
    Reactions,
    CommentsArea,
    Dropdown,
    Dialog,
    Autocomplete,
    UserProfileLink,
    CommentEditor,
    Tooltip,
  },
  resources: {
    discussion() {
      return {
        type: 'document',
        doctype: 'Team Discussion',
        name: this.postId,
        realtime: true,
        whitelistedMethods: {
          trackVisit: 'track_visit',
          closeDiscussion: 'close_discussion',
          reopenDiscussion: 'reopen_discussion',
          moveToProject: {
            method: 'move_to_project',
            validate() {
              if (!this.discussionMoveDialog.project?.value) {
                return 'Project is required to move this discussion'
              }
            },
            onError() {
              this.discussionMoveDialog.show = true
            },
            onSuccess() {
              this.onDiscussionMove()
            },
          },
        },
        onSuccess(doc) {
          this.updateUrlSlug()
          if (
            !this.$route.query.comment &&
            !this.$route.query.fromSearch &&
            doc.last_unread_comment
          ) {
            this.$router.replace({
              query: { comment: doc.last_unread_comment },
            })
          }
          if (this.visitTimer) {
            clearTimeout(this.visitTimer)
            this.visitTimer = null
          }
          this.visitTimer = setTimeout(() => {
            if (
              this.$route.name === 'ProjectDiscussion' &&
              Number(this.$route.params.postId) === doc.name
            ) {
              this.$resources.discussion.trackVisit.submit()
            }
          }, 2000)
        },
      }
    },
  },
  data() {
    return {
      editingContent: false,
      editingTitle: false,
      discussionMoveDialog: {
        show: false,
        project: null,
      },
      showNavbar: false,
    }
  },
  methods: {
    copyLink() {
      let location = window.location
      let url = `${location.origin}${location.pathname}`
      copyToClipboard(url)
    },
    onDiscussionMove() {
      this.$nextTick(() => {
        this.discussionMoveDialog.show = false
        this.discussionMoveDialog.project = null

        this.$router.replace({
          name: 'ProjectDiscussion',
          params: {
            teamId: this.discussion.team,
            projectId: this.discussion.project,
            postId: this.discussion.name,
          },
        })
      })
      this.$resources.discussion.moveToProject.reset()
    },
    handleTitleVisibility(visible, entry) {
      this.showNavbar = !visible
    },
    updateUrlSlug() {
      let doc = this.discussion
      if (!this.$route.params.slug || this.$route.params.slug !== doc.slug) {
        this.$router.replace({
          name: 'ProjectDiscussion',
          params: {
            ...this.$route.params,
            slug: doc.slug,
          },
          query: this.$route.query
        })
      }
    },
  },
  computed: {
    discussion() {
      return this.$resources.discussion.doc
    },
    projectOptions() {
      return teams.data.map((team) => ({
        group: team.title,
        items: getTeamProjects(team.name).map((project) => ({
          label: project.title,
          value: project.name,
        })),
      }))
    },
  },
  pageMeta() {
    if (!this.discussion) return
    let project = this.$getDoc('Team Project', this.discussion.project)
    if (!project) return
    return {
      title: [this.discussion.title, project.title].filter(Boolean).join(' - '),
      emoji: project.icon,
    }
  },
}
</script>
