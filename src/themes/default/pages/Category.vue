<template>
  <div id="category">
    <header class="bg-cl-primary py35 main-container">
      <div class="container-custom-1">
        <breadcrumbs />
        <div class="row middle-sm bg-cl-primary">
          <h1 class="col-sm-12 category-title mb10 align-center">
            <!-- {{ getCurrentCategory.name }} -->
            {{ "Our range" }}
          </h1>
          <div class="sorting col-sm-2 align-right mt50 hidden">
            <label class="mr10">{{ $t('Columns') }}:</label>
            <columns @change-column="columnChange" />
          </div>
          <p class="col-xs-12 col-md-6 m0 pb0 cl-secondary items hidden-xs">
            {{ getCategoryProductsTotal + ' items' }}
          </p>
          <div class="sorting col-sm-2 col-md-6 align-right">
            <label class="mr10">{{ $t('Sort By') }}:</label>
            <sort-by
              :has-label="true"
              @change="changeFilter"
              :value="getCurrentSearchQuery.sort"
            />
          </div>
        </div>
      </div>
      <div class="container-mobile">
        <div class="row m0">
          <p class="col-xs-4 col-md-6 m0 pb0 cl-secondary items items-mobile hidden-md">
            {{ getCategoryProductsTotal + ' items' }}
          </p>
          <button
            class="col-xs-2 mt25 mr15 p15 mobile-filters-button bg-cl-th-accent brdr-none cl-white h5 sans-serif fs-medium-small"
            @click="openFilters"
          >
            {{ $t('Sort By:') }}
          </button>
          <div class="mobile-sorting col-xs-6 mt25">
            <sort-by
              @change="changeFilter"
              :value="getCurrentSearchQuery.sort"
            />
          </div>
        </div>
      </div>
    </header>
    <div class="container-custom pb60">
      <div class="row m0 pt15">
        <div class="col-md-3 start-xs category-filters hidden">
          <sidebar :filters="getAvailableFilters" @changeFilter="changeFilter" />
        </div>
        <div class="col-md-3 start-xs mobile-filters" v-show="mobileFilters">
          <div class="close-container absolute w-100">
            <i class="material-icons p15 close cl-accent" @click="closeFilters">close</i>
          </div>
          <sidebar class="mobile-filters-body" :filters="getAvailableFilters" @changeFilter="changeFilter" />
          <div class="relative pb20 pt15">
            <div class="brdr-top-1 brdr-cl-primary absolute divider w-100" />
          </div>
          <button-full
            class="mb20 btn__filter"
            @click.native="closeFilters"
          >
            {{ $t('Filter') }}
          </button-full>
        </div>
        <div class="col-md-12 px10 border-box products-list">
          <div v-if="isCategoryEmpty" class="hidden-xs">
            <h4 data-testid="noProductsInfo">
              {{ $t('No products found!') }}
            </h4>
            <p>{{ $t('Please change Your search criteria and try again. If still not finding anything relevant, please visit the Home page and try out some of our bestsellers!') }}</p>
          </div>
          <lazy-hydrate :trigger-hydration="!loading" v-if="isLazyHydrateEnabled">
            <product-listing :columns="defaultColumn" :products="getCategoryProducts" />
          </lazy-hydrate>
          <product-listing v-else :columns="defaultColumn" :products="getCategoryProducts" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import LazyHydrate from 'vue-lazy-hydration'
import Sidebar from '../components/core/blocks/Category/Sidebar.vue'
import ProductListing from '../components/core/ProductListing.vue'
import Breadcrumbs from '../components/core/Breadcrumbs.vue'
import SortBy from '../components/core/SortBy.vue'
import { isServer } from '@vue-storefront/core/helpers'
import { Logger } from '@vue-storefront/core/lib/logger'
import { getSearchOptionsFromRouteParams } from '@vue-storefront/core/modules/catalog-next/helpers/categoryHelpers'
import config from 'config'
import Columns from '../components/core/Columns.vue'
import ButtonFull from 'theme/components/theme/ButtonFull.vue'
import { mapGetters } from 'vuex'
import onBottomScroll from '@vue-storefront/core/mixins/onBottomScroll'
import rootStore from '@vue-storefront/core/store';
import { catalogHooksExecutors } from '@vue-storefront/core/modules/catalog-next/hooks'
import { localizedRoute, currentStoreView } from '@vue-storefront/core/lib/multistore'
import { htmlDecode } from '@vue-storefront/core/filters'

const THEME_PAGE_SIZE = 50

const composeInitialPageState = async (store, route, forceLoad = false) => {
  try {
    const filters = getSearchOptionsFromRouteParams(route.params)
    const cachedCategory = store.getters['category-next/getCategoryFrom'](route.path)
    const currentCategory = cachedCategory && !forceLoad ? cachedCategory : await store.dispatch('category-next/loadCategory', { filters })
    const pageSize = store.getters['url/isBackRoute'] ? store.getters['url/getCurrentRoute'].categoryPageSize : THEME_PAGE_SIZE
    await store.dispatch('category-next/loadCategoryProducts', { route, category: currentCategory, pageSize })
    const breadCrumbsLoader = store.dispatch('category-next/loadCategoryBreadcrumbs', { category: currentCategory, currentRouteName: currentCategory.name, omitCurrent: true })

    if (isServer) await breadCrumbsLoader
    catalogHooksExecutors.categoryPageVisited(currentCategory)
  } catch (e) {
    Logger.error('Problem with setting Category initial data!', 'category', e)()
  }
}

export default {
  components: {
    LazyHydrate,
    ButtonFull,
    ProductListing,
    Breadcrumbs,
    Sidebar,
    SortBy,
    Columns
  },
  mixins: [onBottomScroll],
  data () {
    return {
      mobileFilters: false,
      defaultColumn: 3,
      loadingProducts: false,
      loading: true,
      baseUrlImage: ''
    }
  },
  computed: {
    ...mapGetters({
      getCurrentSearchQuery: 'category-next/getCurrentSearchQuery',
      getCategoryProducts: 'category-next/getCategoryProducts',
      getCurrentCategory: 'category-next/getCurrentCategory',
      getCategoryProductsTotal: 'category-next/getCategoryProductsTotal',
      getAvailableFilters: 'category-next/getAvailableFilters'
    }),
    isLazyHydrateEnabled () {
      return config.ssr.lazyHydrateFor.includes('category-next.products')
    },
    isCategoryEmpty () {
      return this.getCategoryProductsTotal === 0
    }
  },
  async asyncData ({ store, route, context }) { // this is for SSR purposes to prefetch data - and it's always executed before parent component methods
    if (context) context.output.cacheTags.add('category')
    await composeInitialPageState(store, route)
  },
  async beforeRouteEnter (to, from, next) {
    if (isServer) next() // SSR no need to invoke SW caching here
    else if (!from.name) { // SSR but client side invocation, we need to cache products and invoke requests from asyncData for offline support
      next(async vm => {
        vm.loading = true
        await composeInitialPageState(vm.$store, to, true)
        await vm.$store.dispatch('category-next/cacheProducts', { route: to }) // await here is because we must wait for the hydration
        vm.loading = false
      })
    } else { // Pure CSR, with no initial category state
      next(async vm => {
        vm.loading = true
        vm.$store.dispatch('category-next/cacheProducts', { route: to })
        vm.loading = false
      })
    }
  },
  methods: {
    openFilters () {
      this.mobileFilters = true
    },
    closeFilters () {
      this.mobileFilters = false
    },
    async changeFilter (filterVariant) {
      this.$store.dispatch('category-next/switchSearchFilters', [filterVariant])
    },
    columnChange (column) {
      this.defaultColumn = column
    },
    async onBottomScroll () {
      if (this.loadingProducts) return
      this.loadingProducts = true
      try {
        await this.$store.dispatch('category-next/loadMoreCategoryProducts')
      } catch (e) {
        Logger.error('Problem with fetching more products', 'category', e)()
      } finally {
        this.loadingProducts = false
      }
    }
  },
  created () {
    this.baseUrlImage = config.server.baseUrl
  },
  // metaInfo () {
  //   const storeView = currentStoreView()
  //   const { meta_title, meta_description, name, slug } = this.getCurrentCategory
  //   const meta = meta_description ? [
  //     { vmid: 'description', name: 'description', content: htmlDecode(meta_description) }
  //   ] : []
  //   /* const categoryLocaliedLink = localizedRoute({
  //     name: 'category-amp',
  //     params: { slug }
  //   }, storeView.storeCode)
  //   const ampCategoryLink = this.$router.resolve(categoryLocaliedLink).href */
  //
  //   return {
  //     // link: [ { rel: 'amphtml', href: ampCategoryLink } ],
  //     title: htmlDecode(meta_title || name),
  //     meta
  //   }
  // }
  metaInfo () {
    const storeView = currentStoreView()
    const {
      meta_title,
      meta_description,
      description,
      cat_banner_desp,
      name,
      slug
    } = this.getCurrentCategory
    // let metaDescriptionCat = this.getCurrentCategory.cat_banner_desp ? this.getCurrentCategory.cat_banner_desp.replace(/<\/?[^>]+(>|$)/g, '') : ''
    let metaLengthCat = 233
    // if (metaDescriptionCat) {
    //   metaDescriptionCat = metaDescriptionCat.length > metaLengthCat ? metaDescriptionCat.substring(0, metaLengthCat - 3) + '...' : metaDescriptionCat
    // }

    const meta_descriptionHtml = htmlDecode(meta_description)
    const meta_descriptionHtmlAfter = meta_descriptionHtml.replace(/<\/?[^>]+(>|$)/g, '')
    const categoryLocaliedLink = localizedRoute(
      {
        name: 'category-amp',
        params: { slug }
      },
      storeView.storeCode
    )
    const ampCategoryLink = this.$router.resolve(categoryLocaliedLink).href
    // const canonicalCategoryLink = this.getCurrentCategory.canonical_url ? this.getCurrentCategory.canonical_url : '/' + this.getCurrentCategory.url_path
    let metaData = [
      {
        property: 'og:url',
        content: 'https://w10.world/costacoffee'
      },
      {
        property: 'og:title',
        content: htmlDecode(meta_title || name)
      },
      {
        property: 'og:type',
        content: 'website'
      },
      {
        property: 'og:description',
        content: meta_description
      }
      // {
      //   property: 'og:image',
      //   content: this.getCurrentCategory.image
      //     ? `${this.baseUrlImage}img/1280/298/resize/catalog/category/${this.getCurrentCategory.image}`
      //     : `/assets/category-images/header.png`
      // }
    ]
    if (meta_description) {
      metaData.push({
        vmid: 'description',
        name: 'description',
        content: meta_descriptionHtmlAfter
      })
    }
    // else {
    //   metaData.push({
    //     vmid: 'description',
    //     name: 'description',
    //     content: metaDescriptionCat
    //   })
    // }
    // const meta = meta_description
    //   ? [
    //     {
    //       vmid: 'description',
    //       name: 'description',
    //       content: meta_descriptionHtmlAfter
    //     }
    //   ]
    //   : []

    return {
      link: [{ rel: 'canonical', href: '/' + this.getCurrentCategory.url_path }],
      title: htmlDecode(meta_title || name),
      titleTemplate: '%s',
      meta: metaData
    }
  }
}
</script>

<style lang="scss" scoped>
  .btn {
    &__filter {
      min-width: 100px;
    }
  }
  .divider {
    width: calc(100vw - 8px);
    bottom: 20px;
    left: -36px;
  }
  .main-container {
    padding: 0 0 33px 0;
  }
  .container-custom {
    padding: 0 79px;
  }
  .container-custom-1 {
    padding: 0 97px;
  }
  .category-filters {
    width: 242px;
  }

  .mobile-filters {
    display: none;
    overflow: auto;
  }

  .mobile-filters-button {
    display: none;
  }

  .mobile-sorting {
    display: none;
  }

  .category-title {
    line-height: 65px;
    font-size: 40px;
    font-weight: 700;
    color: #6E2138;
    font-family: 'Brandon_reg';
    margin: 0;
  }
  .items {
    color: #000000;
    font-weight: 400;
    font-size: 14px;
    margin-top: 4px;
    padding-left: 30px;
  }
  .sorting {
    color: #404042;
    font-weight: 400;
    font-size: 14px;
    margin-top: 4px;
    padding-right: 32px;
    label {
      margin-right: 10px;
    }
  }
  @media (max-width: 767px) {
    .category-title {
      margin: 0;
      font-size: 36px;
      line-height: 40px;
      @media (max-width: 350px) {
        font-size: 27px;
      }
    }
    .container-mobile {
      .row {
        padding: 19px 11px 0 16px;
      }
    }
    .items-mobile {
      display: flex;
      align-items: center;
      margin: 0;
      padding: 0;
    }
    .container-custom {
      padding: 0;
    }
    .products-list {
      width: 100%;
      max-width: none;
    }

    .mobile-filters {
      display: block;
    }

    .mobile-filters-button {
      display: block;
      height: 45px;
      display: block;
      height: 45px;
      background: transparent;
      color: #000;
      margin: 0;
      padding: 0;
    }
    .sorting {
      display: none;
    }

    .mobile-sorting {
      display: block;
      margin: 0;
    }

    .category-filters {
      display: none;
    }
    .product-listing {
      justify-content: center;;
    }

    .mobile-filters {
      position: fixed;
      background-color: #F2F2F2;
      z-index: 5;
      padding: 0 40px;
      left: 0;
      width: 100vw;
      height: 100vh;
      top: 0;
      box-sizing: border-box;
    }

    .mobile-filters-body {
      padding-top: 50px;
    }
  }

  .close-container {
    left: 0;
  }

  .close {
    margin-left: auto;
  }
</style>
<style lang="scss">
.product-image {
  max-height: unset !important;
}
</style>
