<template>
  <div class="flex">
    <DiscussionView
      class="w-full"
      :postId="postId"
      :read-only-mode="Boolean(project.doc.archived_at) || $readOnlyMode"
    />
  </div>
</template>
<script>
import { Avatar, TextEditor } from 'frappe-ui'
import Link from '@/components/Link.vue'
import Reactions from '@/components/Reactions.vue'
import DiscussionList from '@/components/DiscussionList.vue'
import DiscussionView from '@/components/DiscussionView.vue'

export default {
  name: 'ProjectDiscussion',
  props: ['project', 'postId', 'slug'],
  components: {
    TextEditor,
    Avatar,
    Link,
    Reactions,
    DiscussionList,
    DiscussionView,
  },
  methods: {
    isActive(update) {
      return Number(this.$route.params.postId) === update.name
    },
  },
}
</script>
