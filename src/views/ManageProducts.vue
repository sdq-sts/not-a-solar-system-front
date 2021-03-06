<template>
  <v-container grid-list-xl :fluid="screenSize.lgAndDown">
    <v-dialog
      :width="screenSize.xlOnly ? '50%' : '80%'"
      v-model="dialog"
      transition="slide"
      no-click-animation
      persistent
    >
      <product-form
        :productToEdit="productToEdit"
        :clearForm="clearForm"
        :isOpen="dialog"
        :isLoading="loadingForm"
        @cancel="closeDialog"
        @submitEditForm="submitEditForm"
        @submitRegisterForm="submitRegisterForm"
      >
        <UploadFile
          slot="img-upload"
          :clear="clearForm"
          :imgUrl="productImg"
          :defaultImgUrl="defaultProductImg"
          @fileAdded="fileAdded"
        />
      </product-form>
    </v-dialog>

    <v-dialog
      width="400"
      v-model="dialogDelete"
      no-click-animation
      persistent
    >
      <ProductDelete
        :product="productToDelete"
        @cancel="closeDeleteDialog"
        @confirm="confirmDeletion"
      />
    </v-dialog>

    <v-layout row>
      <v-flex
        class="text-xs-right"
        xs12 xl10
        offset-xs0 offset-xl1
      >
        <v-btn
          ref="addProductBtn"
          :small="$vuetify.breakpoint.lgAndDown"
          color="primary"
          class="ma-0"
          @click="registerProduct"
        >{{ registerProductBtnText }}</v-btn>
      </v-flex>
    </v-layout>

    <v-layout row>
      <v-flex
        xs12 xl10
        offset-xs0 offset-xl1>
        <ProductsList
          ref="productsList"
          :loading="loadingProducts"
          :productsList="products"
          @editItem="editProduct"
          @deleteItem="openDeleteDialog"
        />
      </v-flex>
    </v-layout>

    <v-layout row>
      <v-flex class="text-xs-center" xs8 offset-xs2>
        <v-pagination
          ref="productsPag"
          v-if="Math.ceil(productsCount / limit)"
          class="mt-2"
          v-model="page"
          :length="Math.ceil(productsCount / limit)"
          circle
        ></v-pagination>
      </v-flex>
    </v-layout>
  </v-container>
</template>

<script>
import store from '@/store'
import { mapGetters, mapActions } from 'vuex'
import ProductsList from '@/components/Products/ProductsList'
import ProductDelete from '@/components/Products/ProductDelete'
import ProductForm from '@/components/Products/ProductForm'
import UploadFile from '@/components/Shared/UploadFile'
import defaultProductImg from '@/assets/img-not-available.png'

export default {
  components: {
    ProductsList,
    ProductDelete,
    ProductForm,
    UploadFile
  },

  data: () => ({
    defaultProductImg,
    file: null,
    loadingProducts: false,
    loadingForm: false,
    clearForm: false,
    snackbar: false,
    snackbarText: '',
    snackbarColor: '',
    registerProductBtnText: 'Cadastrar Produto',
    successText: 'Produto cadastrado',
    failText: 'Não foi possível cadastrar produto',
    deleteErrorMsg: 'Não foi possível excluir o item',
    noProductMsg: 'Você ainda não tem nenhum produto cadastrado, clique no botão de cadastro e adicione seu primeiro produto.',
    successColor: 'success',
    failColor: 'error',
    dialog: false,
    dialogAdd: false,
    dialogDelete: false,
    dialogEdit: false,
    productToDelete: {},
    productToEdit: null,
    page: 1,
    limit: 12
  }),

  watch: {
    page: {
      handler: 'getProducts'
    }
  },

  computed: {
    ...mapGetters([ 'getImage' ]),
    ...mapGetters('products', [
      'products',
      'productsCount'
    ]),
    screenSize () {
      return this.$vuetify.breakpoint
    },
    productImg () {
      const hasProdutcToEdit = (this.productToEdit && this.productToEdit.image)

      if (hasProdutcToEdit) {
        return this.getImage(this.productToEdit.image)
      } else {
        return ''
      }
    }
  },

  methods: {
    ...mapActions([
      'uploadFile',
      'showSnackbar',
      'requestFileUploadUrl'
    ]),
    ...mapActions('products', [
      'fetchProducts',
      'fetchProductsMeta',
      'deleteProduct',
      'createProduct',
      'updateProduct'
    ]),
    async submitEditForm (payload) {
      const file = this.file
      let product = payload

      this.loadingForm = true

      if (file) {
        try {
          const { data: res } = await this.requestFileUploadUrl({ fileType: file.type, folder: 'products' })
          await this.uploadFile({ url: res.url, file })
          product = { ...payload, image: res.key }
        } catch (error) {
          this.showSnackbar({ color: 'error', text: `Erro ao editar o produto` })
          this.loadingForm = false
        }
      }

      try {
        await this.updateProduct(product)

        this.showSnackbar({ color: 'success', text: `Produto editado com sucesso` })
        this.fetchProductsMeta()
        this.getProducts()
        this.productToEdit = null
        this.loadingForm = false
        this.dialog = false
      } catch (error) {
        this.showSnackbar({ color: 'error', text: `Erro ao editar o produto` })
        this.loadingForm = false
      }
    },
    async submitRegisterForm (payload) {
      const file = this.file
      let product = payload

      this.loadingForm = true

      if (file) {
        const { type: fileType } = file
        const { data: res } = await this.requestFileUploadUrl({ fileType, folder: 'products' })
        await this.uploadFile({ url: res.url, file })
        product = { ...payload, image: res.key }
      }

      try {
        await this.createProduct(product)

        this.fetchProductsMeta()
        this.getProducts()
        this.clearForm = true
        this.productToEdit = null
        this.$nextTick(() => { this.clearForm = false })
        this.showSnackbar({ color: 'success', text: `Produto cadastrado` })
        this.loadingForm = false
      } catch (error) {
        this.showSnackbar({ color: 'danger', text: `Erro ao cadastrar produto` })
        this.loadingForm = false
      }
    },
    async confirmDeletion (payload) {
      try {
        await this.deleteProduct(payload)

        this.fetchProductsMeta()
        this.getProducts()
        this.dialogDelete = false
      } catch (error) {
        this.showSnackbar({ color: 'error', text: 'Erro ao excluir produto' })
      }
    },
    async getProducts () {
      const limit = this.limit
      const page = this.page
      this.loadingProducts = true
      await this.fetchProducts({ page, limit })
      this.loadingProducts = false
    },
    openDeleteDialog (payload) {
      this.productToDelete = payload
      this.dialogDelete = true
    },
    fileAdded (file) {
      this.file = file
    },
    closeDeleteDialog () {
      this.dialogDelete = false
    },
    closeDialog () {
      this.dialog = false
    },
    editProduct (payload) {
      this.productToEdit = payload
      this.dialog = true
    },
    registerProduct () {
      this.productToEdit = null
      this.dialog = true
    },
    animLeaveProductsList () {
      const productsListElm = this.$refs.productsList.$el

      return this.$anime({
        targets: productsListElm,
        translateX: 30,
        opacity: 0,
        easing: 'linear',
        duration: 200
      })
    },
    animAddProductBtn () {
      const addProductBtnElm = this.$refs.addProductBtn.$el

      return this.$anime({
        targets: addProductBtnElm,
        translateY: -30,
        opacity: 0,
        easing: 'linear',
        duration: 350
      })
    },
    animProductsPag () {
      const productsPagElm = this.$refs.productsPag.$el

      return this.$anime({
        targets: productsPagElm,
        translateY: 30,
        opacity: 0,
        easing: 'linear',
        duration: 350
      })
    },
    beforeLeaveAnimations () {
      const animationList = [
        this.animLeaveProductsList(),
        this.animAddProductBtn(),
        this.animProductsPag()
      ]

      const animDurationList = animationList.map(v => v.duration + v.delay)
      const longestAnimDuration = animDurationList.reduce((x, y) => Math.max(x, y))
      const indexOfLongestAnim = animDurationList.indexOf(longestAnimDuration)
      const promiseHandler = (resolve, reject) => {
        animationList[indexOfLongestAnim].complete = resolve
      }

      return new Promise(promiseHandler)
    }
  },

  async beforeRouteEnter (to, from, next) {
    const promises = [
      store.dispatch('products/fetchProductsMeta'),
      store.dispatch('products/fetchProducts', { page: 1, limit: 12 })
    ]

    try {
      await Promise.all(promises)
    } catch (error) {
      store.dispatch('showSnackbar', { color: 'error', text: `Houve um problema ao acessar a página.` })
    }

    next()
  },

  async beforeRouteLeave (to, from, next) {
    // await this.beforeLeaveAnimations()
    next()
  }
}
</script>

<style>
</style>
