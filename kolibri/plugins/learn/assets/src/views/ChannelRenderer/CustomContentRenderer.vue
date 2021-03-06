<template>

  <CoreFullscreen
    ref="html5Renderer"
    class="html5-renderer"
  >
    <div class="iframe-container">
      <iframe
        ref="iframe"
        class="iframe"
        sandbox="allow-scripts allow-same-origin"
        :style="{ backgroundColor: $themePalette.grey.v_100 }"
        frameBorder="0"
        :src="rooturl"
      >
      </iframe>
      <ContentModal
        v-if="overlayIsOpen"
        :contentNode="currentContent"
        :channelTheme="channelTheme"
        @close="overlayIsOpen = false"
      />
    </div>
  </CoreFullscreen>

</template>


<script>

  import urls from 'kolibri.urls';
  import CoreFullscreen from 'kolibri.coreVue.components.CoreFullscreen';
  import Hashi from 'hashi';
  import { now } from 'kolibri.utils.serverClock';
  import { ContentNodeResource, ContentNodeSearchResource } from 'kolibri.resources';
  import router from 'kolibri.coreVue.router';
  import { events, MessageStatuses } from 'hashi/src/hashiBase';
  import { validateTheme } from '../../utils/themes';
  import ContentModal from './ContentModal';

  function createReturnMsg({ message, data, err }) {
    // Infer status from data or err
    const status = data ? MessageStatuses.SUCCESS : MessageStatuses.FAILURE;
    return {
      nameSpace: 'hashi',
      event: events.DATARETURNED,
      data: {
        message_id: message.message_id,
        type: 'response',
        data: data || null,
        err: err || null,
        status,
      },
    };
  }

  export default {
    name: 'CustomContentRenderer',
    components: {
      CoreFullscreen,
      ContentModal,
    },
    props: {
      topic: {
        type: Object,
        required: true,
      },
    },
    data() {
      return {
        overlayIsOpen: false,
        channelTheme: null,
      };
    },
    computed: {
      context() {
        const fetchedEncodedContext = this.$route.query;
        return JSON.stringify(decodeURI(fetchedEncodedContext));
      },
      rooturl() {
        return urls.hashi();
      },
    },
    mounted() {
      this.hashi = new Hashi({ iframe: this.$refs.iframe, now });
      const zipFile = this.topic.files.find(f => f.extension === 'zip');
      this.hashi.initialize({}, {}, zipFile.storage_url, zipFile.checksum);
      this.hashi.on(events.COLLECTIONREQUESTED, message => {
        this.fetchContentCollection(message);
      });
      this.hashi.on(events.MODELREQUESTED, message => {
        this.fetchContentModel.call(this, message);
      });
      this.hashi.on(events.NAVIGATETO, message => {
        this.navigateTo.call(this, message);
      });
      this.hashi.on(events.CONTEXT, message => {
        this.getOrUpdateContext.call(this, message);
      });
      this.hashi.on(events.THEMECHANGED, message => {
        this.updateTheme.call(this, message);
      });
      this.hashi.on(events.SEARCHRESULTREQUESTED, message => {
        this.fetchSearchResult.call(this, message);
      });
      this.hashi.on(events.KOLIBRIVERSIONREQUESTED, message => {
        this.sendKolibriVersion.call(this, message);
      });
    },
    methods: {
      // helper functions for fetching data from kolibri
      // called in mainClient.js

      fetchContentCollection(message) {
        const { options } = message;
        return ContentNodeResource.fetchCollection({
          getParams: {
            ids: options.ids,
            parent: options.parent === 'self' ? this.topic.id : options.parent,
          },
        })
          .then(contentNodes => {
            return createReturnMsg({
              message,
              data: {
                page: options.page ? options.page : 1,
                pageSize: options.pageSize ? options.pageSize : 50,
                results: contentNodes,
              },
            });
          })
          .catch(err => {
            return createReturnMsg({ message, err });
          })
          .then(newMsg => {
            this.hashi.mediator.sendMessage(newMsg);
          });
      },

      fetchContentModel(message) {
        return ContentNodeResource.fetchModel({ id: message.id })
          .then(contentNode => {
            return createReturnMsg({ message, data: contentNode });
          })
          .catch(err => {
            return createReturnMsg({ message, err });
          })
          .then(newMsg => {
            this.hashi.mediator.sendMessage(newMsg);
          });
      },

      fetchSearchResult(message) {
        let searchPromise;
        const { keyword } = message.options;
        if (!keyword) {
          searchPromise = Promise.resolve({
            page: 0,
            pageSize: 0,
            results: [],
          });
        } else {
          searchPromise = ContentNodeSearchResource.fetchCollection({
            getParams: { search: keyword },
          }).then(searchResults => {
            return {
              page: 0,
              pageSize: searchResults.total_results,
              results: searchResults.results,
            };
          });
        }

        return searchPromise
          .then(pageResult => {
            return createReturnMsg({ message, data: pageResult });
          })
          .catch(err => {
            return createReturnMsg({ message, err });
          })
          .then(newMsg => {
            this.hashi.mediator.sendMessage(newMsg);
          });
      },

      navigateTo(message) {
        let id = message.nodeId;
        let context = {};
        return ContentNodeResource.fetchModel({ id })
          .then(contentNode => {
            let routeBase, path;
            if (contentNode && contentNode.kind === 'topic') {
              routeBase = '/topics/t';
              path = `${routeBase}/${id}`;
              router.push({ path: path }).catch(() => {});
            } else if (contentNode) {
              // in a custom context, launch or maintain overlay
              this.currentContent = contentNode;
              this.overlayIsOpen = true;
              context.node_id = contentNode.id;
              context.customChannel = true;
              const encodedContext = encodeURI(JSON.stringify(context));
              router.replace({ query: { context: encodedContext } }).catch(() => {});
            }
            return createReturnMsg({ message, data: {} });
          })
          .catch(err => {
            return createReturnMsg({ message, err });
          })
          .then(newMsg => {
            this.hashi.mediator.sendLocalMessage(newMsg);
          });
      },
      getOrUpdateContext(message) {
        // to update context with the incoming context
        if (message.context) {
          message.context.customChannel = message.context.customChannel || true;
          const encodedContext = encodeURI(JSON.stringify(message.context));
          router.push({ query: { context: encodedContext } }).catch(() => {});
        } else {
          // just return the existing query
          const urlParams = this.$route.query;
          const fetchedEncodedContext = urlParams.has('context')
            ? urlParams.get('context')
            : this.context;
          message.context = decodeURI(JSON.stringify(fetchedEncodedContext));
        }

        const newMsg = createReturnMsg({ message, data: {} });
        this.hashi.mediator.sendLocalMessage(newMsg);
      },
      updateTheme(message) {
        const themeCopy = { ...message };
        delete themeCopy.message_id;
        this.channelTheme = validateTheme(themeCopy);
        const newMsg = createReturnMsg({ message, data: {} });
        return this.hashi.mediator.sendMessage(newMsg);
      },
      sendKolibriVersion(message) {
        const newMsg = createReturnMsg({ message, data: __version });
        return this.hashi.mediator.sendMessage(newMsg);
      },
    },
  };

</script>


<style lang="scss" scoped>

  @import '~kolibri-design-system/lib/styles/definitions';

  .html5-renderer {
    position: relative;
    text-align: center;
  }

  .iframe {
    position: fixed;
    top: 64px;
    left: 0;
    width: 100%;
    height: 100%;
  }

  .iframe-container {
    @extend %momentum-scroll;

    width: 100%;
    overflow: visible;
  }

</style>
