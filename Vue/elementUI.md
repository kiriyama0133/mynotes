-[引言](#引言)

-[初衷](#初衷)





# 引言

​	这里是一条markdown的试验！

```
// modules/my-vuetify-module
export default defineNuxtModule({
  setup(_options, nuxt) {
    // If you're using Nuxt < 3.8.1, you should add a ts-expect-error here
    nuxt.hook('vuetify:registerModule', register => register({
      moduleOptions: {
        /* module specific options */
      },
      vuetifyOptions: {
        /* vuetify options */
      },
    }))
  },
})
```

# 初衷

