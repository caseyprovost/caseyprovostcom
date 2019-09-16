---
layout: post
title:  "Vuex Adventures With SprayPaint"
date:   2019-09-16
categories: software,json,api,rails,ruby
---

Lately, I have been working on a little Vue application to consume various JSON:API APIs. If you have read some of my previous posts you may know I am a huge fan of [Graphiti](https://www.graphiti.dev/guides/). What I hadn't yet played with though is the complimentary JS library called [SprayPaint](https://www.graphiti.dev/js/). Using it is pretty straight forward if you are building a simple Vue application, but if you need to integrate Vuex for a more complex application, and maybe even TypeScript...there are a few discoveries I made that might help you along on your next front-end app journey.

If you don't want to read all this though and just want a repo that shows off how to accomplish all this, then check out [Librarian](https://github.com/caseyprovost/librarian). It is a work in progress, but it is starting to give a good sense for how to use SprayPaint, Vue, Vuex, and TypeScript together :)

## Microservices Introduce Fun!

Now SprayPaint comes with some amazing model helpers like BelongsTo and HasMany that easily accommodates relationships that are part of the same service. But what if you have relationships tied across services? Sadly there is no quick hack to just make this work. Instead, you have to do something as I have done below:

```typescript
@Model()
export class Book extends ApplicationRecord {
  static jsonapiType = 'books'

  foos: string[]

  @Attr() title: string
  @Attr() summary: string
  @Attr() pageCount: number
  @Attr() publicationDate: string
  @Attr() publisherUuid: string
  @Attr() authorUuid: string
  @Attr() createdAt: string
  @Attr() updatedAt: string

  get fullName () {
    return `${this.firstName} ${this.lastName}`
  }

  async author () {
    let { data } = await Author.find(this.authorUuid)
    return data
  }

  async publisher () {
    let { data } = await Publisher.find(this.publisherUuid)
    return data
  }
}
```

This means that you can't call `book.author` as a regular getter method. Instead in returns a promise so you have to do something like:

```typescript
book.author((data) => {
  // data === is the author
  console.log(data)
})
```

Practically speaking it isn't a deal-breaker, but hopefully, in the next week or two, I will have some improvements to share.

## Model-based Mutations With Vuex Require Some TLC

SprayPaint and Vuex are somewhat opposed in how they prefer to handle state. SprayPaint allows mutations on objects in place, but if that object is in a "store" then you have to make some alterations. Take this simple form component for example:

```vue
<template>
  <div class="form-wrapper">
    <div v-if="editableRecord">
      <h2 class="text-2xl mb-4 font-medium">{{ editableRecord.name }}</h2>
      <form @submit.prevent="submitForm" class="w-full">
        <div class="flex flex-wrap -mx-3 mb-6">
          <div class="w-full px-3">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="name">
              Name
            </label>
            <input class="appearance-none block w-full rounded py-3 px-4 mb-2 leading-tight" id="name" type="text" v-model="editableRecord.name">
            <p class="text-red-500 text-xs italic" v-if="hasError('name')">
              {{ errorText('name') }}
            </p>
          </div>
        </div>
        <div class="flex flex-wrap -mx-3 mb-6">
          <div class="w-full md:w-1/2 px-3 mb-6 md:mb-0">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="hometown">
              Hometown
            </label>
            <input class="appearance-none block w-full rounded py-3 px-4 mb-2 leading-tight" id="hometown" type="text" v-model="editableRecord.hometown">
            <p class="text-red-500 text-xs italic" v-if="hasError('hometown')">
              {{ errorText('hometown') }}
            </p>
          </div>
          <div class="w-full md:w-1/2 px-3">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="date_of_birth">
              Born On
            </label>
            <input class="appearance-none block w-full rounded py-3 px-4 leading-tight" id="date_of_birth" type="date" v-model="editableRecord.dateOfBirth">
          </div>
          <div class="flex flex-wrap mt-4">
            <div class="w-full px-3">
              <button class="shadow text-sm bg-teal-700 hover:bg-teal-600 focus:shadow-outline focus:outline-none text-teal-100 font-bold py-2 px-4 rounded" type="submit">
                Save Author
              </button>
            </div>
          </div>
        </div>
      </form>
    </div>
    <div v-if="!editableRecord">
      Loading
    </div>
  </div>
</template>

<script lang="ts">
import { Vue, Component, Prop } from 'vue-property-decorator'
import { Author } from '@/models'
import { mapGetters } from 'vuex'

@Component
export default class EditAuthor extends Vue {
  @Prop() id!: string

  editableRecord = null
  saving = false
  success = false

  mounted () {
    this.syncRecord()
  }

  syncRecord () {
    this.$store.dispatch('authors/fetchRecord', this.id).then(() => {
      this.editableRecord = this.record.dup()
    })
  }

  async submitForm () {
    let success = await this.editableRecord.save()

    if (success) {
      this.syncRecord()
      console.log('success')
    } else {
      console.log('error')
    }
  }

  hasError (field) : boolean {
    if (this.editableRecord && this.editableRecord.errors) {
      return this.editableRecord.errors[field] !== null
    } else {
      return false
    }
  }

  errorText (field) : string {
    if (this.editableRecord.errors && this.editableRecord.errors[field]) {
      return this.editableRecord.errors[field].message
    } else {
      return ''
    }
  }

  get record () : Author | null {
    return this.$store.getters['authors/record']
  }
}
</script>

<style lang="scss" scoped>
  .form-wrapper {
    background: #211C37;
    @apply p-4;
    @apply mt-4;
  }

  input[type="text"],
  input[type="date"] {
    background: #211C37;
    @apply text-gray-500;
    @apply border;
    @apply border-indigo-900;
  }

  input[type="text"]:focus,
  input[type="date"]:focus {
    @apply border-indigo-800;
    @apply outline-none;
  }

  label {
    color: #777CA4;
  }
</style>
```

If you read through the code you will notice the use of `editableRecord`. This is just a local duplicate of the Vuex model in our store. This makes sense because we don't necessarily want the temporary state/modifications to this object to make it back into the store. As long as the component can mutate the object, save it, and handle errors...we are set. The one little gotcha is that when you persist the record, and it succeeds...you need to refetch the remote record since it's state has changed on the server. You can see this being done with the `syncRecord` call in `submitForm`. Overall it works really well, and it might just save you a little bit of time if, like me, you want to build clients to beautiful Graphiti-based APIs.

## Complex Forms Require More Complex Stores

If you recall from earlier, our `Book` model has a `publisher` and an `author`. Let's say you want to build a form where the user can select a new publisher or author from the lists on the servers. You will need to create a module for your store to isolate that data. We wouldn't want a global `publishers` collection, because that is likely used for another page/component. We only want to mutate the list of authors and publishers for our form component. So first let's take a look at the store.

```typescript
import { Module } from 'vuex'
import getters from '@/store/books/getters'
import { actions } from '@/store/books/actions'
import { mutations } from '@/store/books/mutations'
import { BookFormState, RootState } from '@/store/types'

export const state: BookFormState = {
  record: null,
  publishers: [],
  authors: []
}

const namespaced: boolean = true

export const books: Module<BookFormState, RootState> = {
  namespaced,
  state,
  getters,
  actions,
  mutations
}
```

This sets up the state for our form component. You will notice that everything is blank, but that great because we will be loading in these objects when our component is mounted. Speaking of the component... here it is!

```typescript
<template>
  <div class="form-wrapper">
    <div v-if="editableRecord">
      <h2 class="text-2xl mb-4 font-medium">{{ editableRecord.title }}</h2>
      <form @submit.prevent="submitForm" class="w-full">
        <div class="flex flex-wrap -mx-3 mb-6">
          <div class="w-full px-3">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="title">
              Title
            </label>
            <input class="appearance-none block w-full rounded py-3 px-4 mb-2 leading-tight" id="title" type="text" v-model="editableRecord.title">
            <p class="text-red-500 text-xs italic" v-if="hasError('title')">
              {{ errorText('title') }}
            </p>
          </div>
        </div>
        <div class="flex flex-wrap -mx-3 mb-6">
          <div class="w-full md:w-1/2 px-3 mb-6 md:mb-0">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="page_count">
              Page Count
            </label>
            <input class="appearance-none block w-full rounded py-3 px-4 mb-2 leading-tight" id="page_count" type="text" v-model="editableRecord.pageCount">
            <p class="text-red-500 text-xs italic" v-if="hasError('pageCount')">
              {{ errorText('pageCount') }}
            </p>
          </div>
          <div class="w-full md:w-1/2 px-3">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="publication_date">
              Date Published
            </label>
            <input class="appearance-none block w-full rounded py-3 px-4 leading-tight" id="publication_date" type="date" v-model="editableRecord.publicationDate">
          </div>
        </div>
        <div class="flex flex-wrap -mx-3 mb-6">
          <div class="w-full md:w-1/2 px-3 mb-6 md:mb-0">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="author_uuid">
              Author
            </label>
            <v-select label="name" :value="selectedAuthor" :options="authorsForSelect" @input="setSelectedAuthor" class="appearance-none block w-full leading-tight"></v-select>
            <p class="text-red-500 text-xs italic" v-if="hasError('author')">
              {{ errorText('author') }}
            </p>
          </div>
          <div class="w-full md:w-1/2 px-3 mb-6 md:mb-0">
            <label class="block uppercase tracking-wide text-xs font-bold mb-2" for="publisher_uuid">
              Publisher
            </label>
            <v-select label="name" :value="selectedPublisher" :options="publishersForSelect" @input="setSelectedPublisher" class="appearance-none block w-full leading-tight"></v-select>
            <p class="text-red-500 text-xs italic" v-if="hasError('publisher')">
              {{ errorText('publisher') }}
            </p>
          </div>
        </div>
        <div class="flex flex-wrap -mx-3 mb-6">
          <div class="flex flex-wrap mt-4">
            <div class="w-full px-3">
              <button class="shadow text-sm bg-teal-700 hover:bg-teal-600 focus:shadow-outline focus:outline-none text-teal-100 font-bold py-2 px-4 rounded" type="submit">
                Save Book
              </button>
            </div>
          </div>
        </div>
      </form>
    </div>
    <div v-if="!editableRecord">
      Loading
    </div>
  </div>
</template>

<script lang="ts">
import { Vue, Component, Prop } from 'vue-property-decorator'
import { mapGetters } from 'vuex'
import { Book, Author, Publisher } from '@/models'

@Component
export default class EditBook extends Vue {
  @Prop() id!: string

  editableRecord = null
  saving = false
  success = false
  selectedPublisher = null
  selectedAuthor = null

  mounted () {
    this.syncRecord()
  }

  syncRecord () {
    this.$store.dispatch('book/fetchRecord', this.id).then(() => {
      this.editableRecord = this.record.dup()

      this.record.author().then((data) => {
        this.selectedAuthor = data
      })

      this.record.publisher().then((data) => {
        this.selectedPublisher = data
      })

      this.$store.dispatch('book/fetchPublishers')
      this.$store.dispatch('book/fetchAuthors')
    })
  }

  loadPublishers () {
    this.$store.dispatch('publishers/fetchcollection', this.id).then((collection) => {
      this.publishers = this.sortRecordsByAttriibute(collection, 'name')
    })
  }

  loadAuthors () {
    this.$store.dispatch('authors/fetchcollection', this.id).then((collection) => {
      this.authors = this.sortRecordsByAttriibute(collection, 'name')
    })
  }

  setSelectedPublisher (value) {
    this.selectedPublisher = this.publishers.find(p => p.id === value.id)
    this.editableRecord.publisherUuid = value.id
  }

  setSelectedAuthor (value) {
    this.editableRecord.authorUuid = value.id
    this.selectedAuthor = this.authors.find(p => p.id === value.id)
  }

  sortRecordsByAttriibute (records, attribute) : [] {
    return records.sort((a, b) => {
      // Use toUpperCase() to ignore character casing
      const attributeA = a[attribute].toUpperCase()
      const attributeB = b[attribute].toUpperCase()

      let comparison = 0

      if (attributeA > attributeB) {
        comparison = 1
      } else if (attributeA < attributeB) {
        comparison = -1
      }

      return comparison
    })
  }

  async submitForm () {
    let success = await this.editableRecord.save()

    if (success) {
      this.syncRecord()
      console.log('success')
    } else {
      console.log('error')
    }
  }

  hasError (field) : boolean {
    if (this.editableRecord.errors) {
      return this.editableRecord.errors[field] !== null
    } else {
      return false
    }
  }

  errorText (field) : boolean {
    if (this.editableRecord.errors && this.editableRecord.errors[field]) {
      return this.editableRecord.errors[field].message
    } else {
      return ''
    }
  }

  get publishersForSelect () : [] {
    if (this.publishers.length === 0) { return [] } else {
      return this.publishers.map((publisher) => {
        return {
          id: publisher.id,
          name: publisher.name
        }
      })
    }
  }

  get authorsForSelect () : [] {
    if (this.authors.length === 0) { return [] } else {
      return this.authors.map((author) => {
        return {
          id: author.id,
          name: author.name
        }
      })
    }
  }

  get record () : Book | null {
    return this.$store.getters['book/record']
  }

  get authors () : Author[] {
    return this.$store.getters['book/authors']
  }

  get publishers () : Publisher[] {
    return this.$store.getters['book/publishers']
  }
}
</script>

<style lang="scss" scoped>
  .form-wrapper {
    background: #211C37;
    @apply p-4;
    @apply mt-4;
  }

  input[type="text"],
  input[type="date"],
  select {
    background: #211C37;
    @apply text-gray-500;
    @apply border;
    @apply border-indigo-900;
  }

  input[type="text"]:focus,
  input[type="date"]:focus,
  select:focus {
    @apply border-indigo-800;
    @apply outline-none;
  }

  label {
    color: #777CA4;
  }
</style>
```

You will notice that the component only tracks the book state. It doesn't care about the author or publisher or how to load the selected relationships because we took care of that in the `Book` model above. This is pretty slick in my opinion and isolates the component's concerns.

## Use TypeScript!

With your Vue components, there is a lot of ceremony with props, data, methods, and computed methods. If you use `Vue.extend({})` to create your components you will surely notice this. However, if you use class components you get a whole lot for free. You will notice that in the shared components there is no hash of methods or computed methods and prop definitions are a single line with `@Prop`. Not only that, but you will get all kinds of wonderful errors if your props, data, or methods return results that do not match up with the defined types. This will save you endless minutes and maybe even hours tracking down obscure prop-drilling errors.

## Don't Use Microservices

If you are building a complex front-end against multiple services... spend some time thinking about that. If you want a first-class experience on the front-end then use a proxy instead. I have started a Ruby one [here](https://github.com/caseyprovost/spider). Now it is very specific to my little learning project...but it does showcase a bit of how to pull multiple microservices together into a single GraphQL proxy to make the lives of JS-slinging folks much easier.

That's all for now! Thanks so much for taking the time to read through this. If you think I am totally off my rocker or just want to share some improvements feel free to drop a comment below or even open an issue on one of the projects I shared with you today. I'll be continuing to develop them and hope to pass those learnings to you. Cheers!
