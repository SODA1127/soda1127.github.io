---
layout: page
title: Contact
description: Let's talk.
permalink: /contact/
---

<style type="text/css" media="screen">
  .container {
    margin: 0px auto;
    max-width: 600px;
  }
</style>

<div class="container">

  <h2>{{ site.translations.contact.title | default: "Talk to me" }}</h2>

  <div id="form" class="contact-form">
    <form accept-charset="UTF-8" method="POST" action="https://formspree.io/{{ site.email }}" v-on:submit.prevent="validateBeforeSubmit" ref="contact">
      <fieldset>
        <input type="hidden" name="_subject" value="{{ site.translations.contact.subject | default: 'New contact!' }}" />
        <input type="hidden" name="_next" value="{{ site.url }}{{ site.sent_message_url }}" />
        <input type="hidden" name="_language" value="{{ site.language }}" />

        <input type="text" name="name" placeholder="{{ site.translations.contact.placeholders.name | default: 'Your name' }}" v-validate="'required'"
               :class="{ 'has-error': errors.has('name') }">
        <span v-if="errors.has('name')" v-cloak>${ errors.first('name') }</span>

        <input type="text" name="email" placeholder="{{ site.translations.contact.placeholders.email | default: 'Your email' }}" v-validate="'required|email'"
               :class="{ 'has-error': errors.has('email') }">
        <span v-if="errors.has('email')" v-cloak>${ errors.first('email') }</span>

        <textarea name="message" onkeyup="adjust_textarea(this)" placeholder="{{ site.translations.contact.placeholders.message | default: 'Your message' }}" v-validate="'required'"
                  :class="{ 'has-error': errors.has('message') }"></textarea>
        <span v-if="errors.has('message')" v-cloak>${ errors.first('message') }</span>

        <button type="submit">{{ site.translations.contact.submit_btn | default:  "Send" }}</button>
      </fieldset>
    </form>
  </div>

</div>


<script type="text/javascript">
function adjust_textarea(h) {
    h.style.height = "200px";
    h.style.height = (h.scrollHeight)+"px";
}
</script>

<script src="https://unpkg.com/vue@2.4.2"></script>
<script src="https://unpkg.com/vee-validate@2.0.0-rc.8"></script>
<script type="text/javascript">
Vue.use(VeeValidate);
new Vue({
  el: '#form',
  delimiters: ['${', '}'],
  methods: {
    validateBeforeSubmit: function () {
      this.$validator.validateAll();
      if (!this.errors.any()) {
        this.$refs.contact.submit();
      }
    }
  }
});
</script>
