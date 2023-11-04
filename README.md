# uniapp-canvas-cutImg
vue2
<template>
  <Imagecanvas :Imageurl="imgurl" @cutwidthorheightnotenough="widthorheightnotenough" @imagesavesuccess="imagesavesuccess" @imagecanvasback="imagecanvasback" :condition="Imagecondition"></Imagecanvas>
</template>

<script>
export default {
    data(){
        return{
            imgurl:'',
            Imagecondition: {}
        }
    },
    methods:{
        widthorheightnotenough(){},
        imagesavesuccess(){},
        imagecanvasback(){},
    }
}
</script>
