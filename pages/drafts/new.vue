<template>
  <div>
    <section class="section mt-5">
      <div class="container is-max-desktop">
        <h1 class="title"><font-awesome-icon icon="pen-nib" /> 趣味を書く</h1>
        <h2 class="subtitle">自分の好きな趣味をみんなに広めるのだぁ</h2>
        <div class="field">
          <label class="label"
            >タイトル<span style="color: #f38181"> *</span></label
          >
          <div class="control">
            <input v-model="title" class="input" placeholder="記事のタイトル" />
          </div>
        </div>
        <div class="field">
          <div class="level mb-0">
            <label class="label"
              >タグ<span style="color: #f38181"> *</span></label
            >
            <button
              class="button is-rounded is-small is-primary"
              @click="addTagSuggestBox()"
            >
              タグを追加
            </button>
          </div>
          <div class="control">
            <div class="columns is-multiline">
              <div
                v-for="(searchText, index) in searchTexts"
                :key="'searchText-' + index"
                class="column has-text-right is-6"
              >
                <div class="level">
                  <input
                    v-model="searchTexts[index]"
                    class="input"
                    placeholder="記事に関するタグ"
                    name="yourarea"
                    autocomplete="on"
                    list="suggestList"
                  />
                  <a
                    v-show="searchTexts.length > 1"
                    class="delete ml-2"
                    @click="deleteTagSuggestBox(index)"
                  ></a>
                </div>
              </div>
              <datalist id="suggestList">
                <option v-for="(n, index) in tagSuggestions" :key="n + index">
                  {{ n }}
                </option>
              </datalist>
            </div>
          </div>
        </div>
        <div class="field">
          <label class="label">書いた人</label>
          <div class="control">
            <input
              v-model="author"
              class="input"
              placeholder="書いた人のなまえ"
            />
          </div>
        </div>
        <div class="field">
          <label class="label"
            >本文<span style="color: #f38181"> *</span></label
          >
          <div class="control">
            <mavon-editor
              v-model="content"
              :toolbars="markdownOption"
              language="ja"
              placeholder="記事を書くのだぁ！マークダウン形式が使えるのだぁ！"
              class="mavon-editor"
            />
          </div>
        </div>
        <div class="columns">
          <div class="column has-text-right">
            <button type="button" class="button is-primary" @click="submit">
              投稿する！
            </button>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import "mavon-editor/dist/css/index.css";

export default {
  async asyncData({ query, $getTag, $getSuggestions }) {
    let recommendedTagName = "";
    if (typeof query.tagId !== "undefined") {
      // クエリパラメータにtagIdが渡されていたら、そのタグを推薦
      const recommendedTag = await $getTag(query.tagId);
      recommendedTagName = recommendedTag.name;
    }
    let data = await $getSuggestions();
    return {
      recommendedTagName: recommendedTagName,
      tagSuggestions: data.tagSuggestions,
    };
  },
  data() {
    return {
      title: "",
      content: "",
      tags: [],
      innerSearchText: "",
      author: "",
      searchTexts: [""],
      markdownOption: {
        bold: true,
        italic: true,
        header: true,
        underline: true,
        strikethrough: true,
        mark: true,
        superscript: true,
        subscript: true,
        quote: true,
        ol: true,
        ul: true,
        link: true,
        imagelink: true,
        code: true,
        table: true,
        fullscreen: false,
        htmlcode: true,
        help: true,
      },
    };
  },

  created() {
    // 直前に見ていた趣味のタグをプリセットする
    this.searchTexts[0] = this.recommendedTagName;
  },

  methods: {
    addTagSuggestBox() {
      this.searchTexts.push("");
    },

    deleteTagSuggestBox(index) {
      if (this.searchTexts.length > 1) {
        this.searchTexts.splice(index, 1);
      }
    },

    submit: async function () {
      let timestamp = await this.$getFirebaseTimestamp();
      let article = {
        id: null,
        title: this.title,
        body: this.content,
        author: this.author ? this.author : "ほびー",
        tags: [],
        createdAt: timestamp,
        updatedAt: timestamp,
      };

      if (
        this.searchTexts[0] === "" ||
        this.title === "" ||
        this.content === ""
      ) {
        alert("タイトル・タグ・本文は必須項目です");
        return;
      }

      if (!confirm(`以下の内容で投稿しますか？\n${this.title}`)) {
        return;
      }

      const validatedSearchText = this.searchTexts
        .map((text) => text.trim())
        .filter((x, i, self) => {
          return self.indexOf(x) === i;
        });
      for await (let searchText of validatedSearchText) {
        let existingTag = await this.$getExistingTag(searchText);
        if (existingTag == null) {
          // 新規登録
          let newTagSuggestions = this.tagSuggestions;
          newTagSuggestions.push(searchText);
          let documentRefId = await this.$createTag(searchText);
          await this.$addTagSuggestions(newTagSuggestions);
          article.tags.push(documentRefId);
          console.debug("tag pushed :", article.tags);
        } else {
          article.tags.push(existingTag.id);
          // tagのvolumeを増やす
          console.debug("tag pushed :", article.tags);
          this.$incrementArticlesCount(existingTag.id, 1);
        }
      }

      console.debug("debug1 tags : ", article.tags);
      console.debug("debug1.1 : ", article);
      const transitionArticleId = await this.$registerArticle(article);
      const transitionTagId = article.tags[0];
      const pagePath = `/${transitionTagId}/${transitionArticleId}`;

      alert("記事を投稿しました!");
      this.$router.push(pagePath);
    },
  },
};
</script>

<style>
.center {
  margin-left: auto;
  margin-right: auto;
}

.mavon-editor {
  max-height: 1vh;
}

.v-note-show {
  font-family: heisei-maru-gothic-std, Meiryo, sans-serif;
}
</style>
