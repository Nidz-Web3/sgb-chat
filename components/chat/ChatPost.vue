<template>
<div class="card mb-3 border" v-if="post">
  <div class="card-body row">
    <div class="col-2 col-md-1">
      <NuxtLink :to="'/profile/?id='+String(showDomainOrFullAddress)">
        <ProfileImage 
          class="img-fluid rounded-circle"
          :address="authorAddress" 
          :domain="authorDomain"
          :image="getOrbisImage"
        />
      </NuxtLink>
    </div>

    <div class="col-10 col-md-11">
      
      <!-- post author and timestamp -->
      <p class="card-subtitle mb-2 text-muted">
        <NuxtLink class="link-without-color hover-color" :to="'/profile/?id='+String(showDomainOrFullAddress)">{{showDomainOrAddressOrAnon}}</NuxtLink>
        <span v-if="post.timestamp"> · <NuxtLink class="link-without-color hover-color" :to="'/post/?id='+post.stream_id">{{timeSince}}</NuxtLink></span>
      </p>

      <!-- post text -->
      <div @click="openPostDetails">
        <p
          class="card-text text-break"
          v-if="parsedText.length > postLengthLimit && !showFullText"
        >
          <span  
            v-html="parsedText.substring(0, postLengthLimit) + ' ... '">
          </span>
          <span class="cursor-pointer hover-color" @click="showFullText = true">Read more</span>
        </p>

        <p v-if="parsedText.length < postLengthLimit || showFullText" class="card-text text-break" v-html="parsedText"></p>
      </div>

      <!-- quoted post (replied) -->
      <ChatQuote class="mt-3" :post="quotePost" v-if="showQuote" />

      <!-- post actions -->
      <p class="card-subtitle mt-3 text-muted">
        
        <span>
          <i @click="likePost" class="cursor-pointer hover-color" :class="alreadyLiked ? 'bi bi-heart-fill text-primary' : 'bi bi-heart'"></i> 
          {{post.count_likes}}
        </span>

        <NuxtLink v-if="!post.master" class="ms-3 link-without-color hover-color" :to="'/post/?id='+post.stream_id">
          <i class="bi bi-chat"></i> 
          {{post.count_replies}}
        </NuxtLink>

        <IggyPostMint :post="post" :parsedText="parsedText" />

        <span v-if="post.master" class="cursor-pointer hover-color ms-2" data-bs-toggle="modal" :data-bs-target="'#replyModal'+post.stream_id">
          <i class="bi bi-reply" /> Reply
        </span>

        <span v-if="isCurrentUserAuthor" class="cursor-pointer hover-color ms-2" data-bs-toggle="modal" :data-bs-target="'#deleteModal'+post.stream_id">
          <i class="bi bi-trash" /> Delete
        </span>
      </p>
    </div>
  </div>


  <!-- Reply Modal -->
  <div class="modal fade" :id="'replyModal'+post.stream_id" tabindex="-1" :aria-labelledby="'replyModalLabel'+post.stream_id" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" :id="'replyModalLabel'+post.stream_id">Reply to comment</h1>
          <button type="button" class="btn-close" :id="'closeReplyModal'+post.stream_id" data-bs-dismiss="modal" aria-label="Close"></button>
        </div>
        <div class="modal-body">

          <div v-if="!userStore.getIsConnectedToOrbis">
            <p>First connect to Ceramic before you can reply:</p>

            <button class="btn btn-primary" @click="connectToOrbis">Connect to Ceramic</button>
          </div>

          <div class="form-group mt-2 mb-2" v-if="userStore.getIsConnectedToOrbis">
            <textarea 
              v-model="replyText"  
              class="form-control" 
              rows="3" 
              placeholder="Enter your reply"
            ></textarea>
          </div>
        </div>

        <div class="modal-footer">
          <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
          <button :disabled="!userStore.getIsConnectedToOrbis" type="button" class="btn btn-primary" @click="replyPost">Submit reply</button>
        </div>
      </div>
    </div>
  </div>
  <!-- END Reply Modal -->

  <!-- Delete Modal -->
  <div class="modal fade" :id="'deleteModal'+post.stream_id" tabindex="-1" :aria-labelledby="'deleteModalLabel'+post.stream_id" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" :id="'deleteModalLabel'+post.stream_id">Delete post</h1>
          <button type="button" class="btn-close" :id="'closeDeleteModal'+post.stream_id" data-bs-dismiss="modal" aria-label="Close"></button>
        </div>
        <div class="modal-body">
          Do you really want to delete this post?
        </div>
        <div class="modal-footer">
          <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
          <button type="button" class="btn btn-primary" @click="deletePost" :disabled="waitingDeletePost">
            <span v-if="waitingDeletePost" class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>
            Delete
          </button>
        </div>
      </div>
    </div>
  </div>
  <!-- END Delete Modal -->

</div>
</template>

<script>
import { ethers } from 'ethers';
import sanitizeHtml from 'sanitize-html';
import { useEthers, shortenAddress } from 'vue-dapp';
import ResolverAbi from "~/assets/abi/ResolverAbi.json";
import resolvers from "~/assets/data/resolvers.json";
import { useToast } from "vue-toastification/dist/index.mjs";
import { useUserStore } from '~/store/user';
import ProfileImage from "~/components/profile/ProfileImage.vue";
import IggyPostMint from "~~/components/minted-posts/IggyPostMint.vue";
import ChatQuote from "~/components/chat/ChatQuote.vue";
import { imgParsing, imgWithoutExtensionParsing, urlParsing, youtubeParsing } from '~/utils/textUtils';

export default {
  name: "ChatPost",
  emits: ["insertReply", "removePost"],
  props: ["post", "showQuotedPost"],

  components: {
    ChatQuote,
    ProfileImage,
    IggyPostMint
  },

  data() {
    return {
      alreadyLiked: false,
      authorAddress: null,
      authorDomain: null,
      parsedText: null,
      postLengthLimit: 550,
      quotePost: null,
      quoteLimit: 200,
      replyText: null,
      showFullText: false,
      waitingDeletePost: false
    }
  },

  created() {
    if (this.post.creator_details.metadata) {
      this.fetchAuthorDomain();
    }

    this.parsePostText();

    if (
      this.route.href === "/post/?id=" + this.post.stream_id ||
      this.route.href === "/post?id=" + this.post.stream_id
    ) {
      this.showFullText = true;
    }

    // create quote post object
    if (this.post.reply_to_details) {
      this.quotePost = {
        body: this.post.reply_to_details.body,
        stream_id: this.post.reply_to,
      }

      if (this.post.reply_to_creator_details.profile && this.post.reply_to_creator_details.profile.pfp) {
        this.quotePost["pfp"] = this.post.reply_to_creator_details.profile.pfp;
      }

      if (this.post.reply_to_creator_details.profile && this.post.reply_to_creator_details.profile.username) {
        this.quotePost["username"] = this.post.reply_to_creator_details.profile.username;
      }

      if (this.post.reply_to_creator_details.metadata && this.post.reply_to_creator_details.metadata.address) {
        this.quotePost["address"] = this.post.reply_to_creator_details.metadata.address;
      }
    }
  },

  computed: {
    getOrbisContext() {
      if (this.post?.context) {
        return this.post.context;
      } else if (this.post?.content.context) {
        return this.post.content.context;
      } else if (this.post?.context_details.context_id) {
        return this.post.context_details.context_id;
      } else {
        return this.$config.orbisContext;
      }
    },

    getOrbisImage() {
      if (this.post.creator_details.profile) {
        return this.post.creator_details.profile.pfp;
      }

      return null;
    },

    isCurrentUserAuthor() {
      return String(this.authorAddress).toLowerCase() === String(this.address).toLowerCase();
    },

    isUserConnectedOrbis() {
      return this.userStore.getIsConnectedToOrbis;
    },

    showDomainOrAddressOrAnon() {
      if (this.authorDomain) {
        return this.authorDomain;
      } else if (this.post.creator_details.metadata) {
        return this.shortenAddress(this.post.creator_details.metadata.address);
      } else {
        return "Anon";
      }
    },

    showDomainOrFullAddress() {
      if (this.authorDomain) {
        return this.authorDomain;
      } else if (this.post.creator_details.metadata) {
        return this.post.creator_details.metadata.address;
      }

      return null;
    },

    showQuote() {
      if (this.post.master && this.post.master !== this.post.reply_to) {
        return true;
      }

      if (this.post.master && this.showQuotedPost) {
        return true;
      }

      return false;
    },

    timeSince() {
      if (!isNaN(this.post.timestamp)) {
        const timePosted = new Date(this.post.timestamp * 1000);
        const now = new Date();
        const secondsPast = (now.getTime() - timePosted.getTime() ) / 1000;

        if (secondsPast < 60) return 'now';
        if (secondsPast < 3600) return parseInt(secondsPast/60) + 'min';
        if (secondsPast <= 86400) return parseInt(secondsPast/3600) + 'h';
        if (secondsPast <= 2628000) return parseInt(secondsPast/86400) + 'd';
        if (secondsPast <= 31536000) return parseInt(secondsPast/2628000) + 'mo';
        if (secondsPast > 31536000) return parseInt(secondsPast/31536000) + 'y';
      }

      return null;
    }
  },

  methods: {

    async checkIfAlreadyLiked() {
      // check if user has already liked this post
      if (this.userStore.getIsConnectedToOrbis) {
        let res = await this.$orbis.getReaction(
          String(this.post.stream_id), 
          String(this.userStore.getDidParent) // current user's did
        );

        /** Check if request is successful or not */
        if (res.status == 200) {
          if (res.data && res.data.type === "like") {
            this.alreadyLiked = true; // mark as liked
          }
        }
      }
    },

    async connectToOrbis() {
      let res = await this.$orbis.connect_v2({
        provider: this.signer.provider.provider, 
        lit: false
      });

      /** Check if connection is successful or not */
      if(res.status == 200) {
        this.userStore.setIsConnectedToOrbis(true);

        if (this.$orbis.session) {
          this.userStore.setDid(this.$orbis.session.did._id);
          this.userStore.setDidParent(this.$orbis.session.did._parentId);
        }
      } else {
        console.log("Error connecting to Ceramic: ", res);
        this.toast(res.result, {type: "error"});
      }
    },

    async deletePost() {
      if (this.userStore.getIsConnectedToOrbis) {
        this.waitingDeletePost = true;

        let res = await this.$orbis.deletePost(
          String(this.post.stream_id)
        );

        /** Check if request is successful or not */
        if (res.status == 200) {
          this.toast("Post deleted successfully", {type: "success"});
          this.$emit("removePost", this.post.stream_id);
          document.getElementById('closeDeleteModal'+this.post.stream_id).click();
          this.waitingDeletePost = false;
        } else {
          this.toast("Error deleting post", {type: "error"});
          console.log("Error posting via Orbis to Ceramic: ", res);
          this.waitingDeletePost = false;
        }
      } else {
        this.toast("Please sign into chat to be able to delete your post.", {type: "error"});
        this.waitingDeletePost = false;
      }
    },

    async fetchAuthorDomain() {
      // find out if post author has a domain name
      this.authorAddress = this.post.creator_details.metadata.address;

      if (this.authorAddress) {
        // check session storage if author's domain is already stored
        const storedDomain = sessionStorage.getItem(String(this.authorAddress).toLowerCase());

        if (storedDomain) {
          this.authorDomain = storedDomain;
        } else {
          // fetch provider from hardcoded RPCs
          let provider = this.$getFallbackProvider(this.$config.supportedChainId);

          if (this.isActivated && this.chainId === this.$config.supportedChainId) {
            // fetch provider from user's MetaMask
            provider = this.signer;
          }

          const contract = new ethers.Contract(resolvers[this.$config.supportedChainId], ResolverAbi, provider);

          // get author's default domain
          const domainName = await contract.getDefaultDomain(
            String(this.authorAddress).toLowerCase(), 
            String(this.$config.tldName).toLowerCase()
          );

          if (domainName) {
            this.authorDomain = domainName + this.$config.tldName;
            sessionStorage.setItem(String(this.authorAddress).toLowerCase(), this.authorDomain);
          } 
        }
      }
    },

    async likePost() {
      if (this.userStore.getIsConnectedToOrbis && !this.alreadyLiked) {
        // mark as liked
        this.alreadyLiked = true;
        this.post.count_likes++;

        // like the post
        let res = await this.$orbis.react(
          this.post.stream_id,
          "like"
        );

        /** Check if request is successful or not */
        if (res.status !== 200) {
          // if failed request, unmark as liked
          this.alreadyLiked = false;
          this.post.count_likes--;
          console.log("Error liking the post: ", res);
          this.toast(res.result, {type: "error"});
        }
      } else if (this.userStore.getIsConnectedToOrbis && this.alreadyLiked) {
        // un-mark as liked
        this.alreadyLiked = false;
        this.post.count_likes--;

        // remove reaction ("un-like" the post)
        let res = await this.$orbis.react(
          this.post.stream_id,
          "none" // "none" removes the previous "like" reaction
        );

        /** Check if request is successful or not */
        if (res.status !== 200) {
          // if failed request, mark as liked again
          this.alreadyLiked = true;
          this.post.count_likes++;
          console.log("Error un-liking the post: ", res);
          this.toast(res.result, {type: "error"});
        }
      } else {
        // @todo: open a modal to sign into chat instead
        await this.connectToOrbis();

        this.toast("Signed into chat, now please try to like the post again.", {type: "success"});
      }
    },

    async replyPost() {
      if (this.userStore.getIsConnectedToOrbis) {

        const options = {
          master: this.post.master, // the main post in the thread
          reply_to: this.post.stream_id, // important: reply_to needs to be filled out even if the reply is directly to the master post
          body: this.replyText, 
          context: this.getOrbisContext
        }

        // if post has tags, add them to the options
        if (this.post?.content?.tags) {
          options["tags"] = this.post.content.tags;
        }

        // post on Orbis & Ceramic
        let res = await this.$orbis.createPost(options);

        /** Check if posting is successful or not */
        if (res.status == 200) {
          // success
          this.toast("Reply successfully posted", {type: "success"});
          this.$emit("insertReply", res.doc, this.post.stream_id, this.replyText, this.parsedText, this.post.creator_details.metadata.address);
          document.getElementById('closeReplyModal'+this.post.stream_id).click();
          this.replyText = null;
        } else {
          console.log("Error posting via Orbis to Ceramic: ", res);
          this.toast(res.result, {type: "error"});
        }

      } else {
        this.toast("Please sign into chat to be able to reply to a post.", {type: "error"});
      }
    },

    openPostDetails() {
      // navigate to post details page if you're not already there
      /*
      if (
        this.route.path !== "/post/" && // no post on any /post page is clickable
        this.route.path !== "/post" && // no post on any /post page is clickable
        this.route.href !== "/post?id=" + this.post.stream_id &&
        this.route.href !== "/post/?id=" + this.post.stream_id
      ) {
        console.log("navigate to post details page")
        this.$router.push({ name: 'post', query: { id: this.post.stream_id } });
      }
      */

      this.$router.push({ name: 'post', query: { id: this.post.stream_id } });
    },

    parsePostText() {
      let postText = this.post.content.body;

      postText = sanitizeHtml(postText, {
        allowedTags: [ 'li', 'ul', 'ol', 'br', 'em', 'strong', 'i', 'b' ],
        allowedAttributes: {}
      });

      postText = imgParsing(postText);
      postText = imgWithoutExtensionParsing(postText);
      postText = youtubeParsing(postText);
      postText = urlParsing(postText);

      this.parsedText = postText.replace(/(\r\n|\n|\r)/gm, "<br/>");
    }
  },

  setup() {
    const route = useRoute();
    const { address, chainId, isActivated, signer } = useEthers();
    const toast = useToast();
    const userStore = useUserStore();

    return { address, chainId, isActivated, route, shortenAddress, signer, toast, userStore }
  },

  watch: {
    isUserConnectedOrbis(newVal, oldVal) {
      if (newVal) {
        this.checkIfAlreadyLiked();
      }
    }
  }
}
</script>
