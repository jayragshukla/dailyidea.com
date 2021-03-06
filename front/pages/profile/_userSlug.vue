<template>
  <users-profile
    :initial-profile-data="userInfo"
    :ideas="userIdeas"
    :load-more-ideas-is-possible="loadMoreIdeasIsPossible"
  ></users-profile>
</template>

<script>
import UsersProfile from '@/components/profilePage/UsersProfile'
import userInfoBySlug from '@/graphql/query/userInfoBySlug'
import getUsersIdeas from '@/graphql/query/getUsersIdeas'
import loadIdeas from '@/components/ideasList/loadIdeas'
import getIdeas from '@/graphql/query/getIdeas'

export default {
  name: 'UserSlugVue',
  components: { UsersProfile },

  async asyncData({ app, route, store, error }) {
    const userSlug = route.params.userSlug
    const isMyProfile = store.getters['userData/slug'] === route.params.userSlug
    const userInfoRequest = app.$amplifyApi.graphql({
      query: userInfoBySlug,
      variables: { slug: userSlug },
      authMode: isMyProfile ? undefined : 'API_KEY'
    })
    const userIdeasRequest = loadIdeas(
      app.$amplifyApi,
      isMyProfile ? 'ideas' : 'getUsersIdeas',
      isMyProfile ? getIdeas : getUsersIdeas,
      isMyProfile ? { limit: 25 } : { authorSlug: userSlug, limit: 25 },
      isMyProfile ? undefined : 'API_KEY'
    )
    const [userInfoResponse, userIdeasResponse] = await Promise.all([
      userInfoRequest,
      userIdeasRequest
    ])
    const userInfo = userInfoResponse.data.userInfoBySlug.userInfo
    if (!userInfo) {
      error({ statusCode: 404, message: 'User not found' })
    }
    const userIdeas = userIdeasResponse.ideas
    const loadMoreIdeasIsPossible = !!userIdeasResponse.nextToken
    return {
      userInfo,
      userIdeas,
      loadMoreIdeasIsPossible
    }
  },

  head() {
    return {
      meta: [
        {
          hid: 'og:title',
          property: 'og:title',
          content: `See ${this.userInfo.name}'s ideas on Daily Idea`
        }
      ]
    }
  }
}
</script>

<style scoped></style>
