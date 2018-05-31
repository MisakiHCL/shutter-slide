<template>
  <div class="shutterSlide" :style="{ width:canvasWidth+'px',height:canvasHeight+'px' }"></div>
</template>

<script>
import * as PIXI from 'pixi.js'
import TweenMax from "greensock"

export default {
  data () {
    return {
      canChange: true,
      activeIndex: 0,
      countNum: 8,
      imgData:[],
      canvasWidth: 1208,
      canvasHeight: 750,
      wheel:false,
      scrollY:false
    }
  },
  props: {
    shutterOptions: {
      type: Object,
      required: true,
      default: () => ({})
    }
  },
  created(){
    for(const options in this.shutterOptions) {
      if (this[options]!=undefined&&this.shutterOptions[options]!=undefined) {
        this[options] = this.shutterOptions[options];
      }
    }
  },
  mounted () {
    const renderer = PIXI.autoDetectRenderer(this.canvasWidth, this.canvasHeight, {antialias: true});
    if(this.scrollY){
      renderer.plugins.interaction.autoPreventDefault = false;
      renderer.view.style['touch-action'] = 'auto';
    }
    const stage = new PIXI.Container(),Sprite = PIXI.Sprite,Rectangle = PIXI.Rectangle;
    let disX = 0,getX = 0,getY = 0,imgLoaded = 0,getKey = [],imgList1=[],imgList2=[],maskList1 = [],maskList2 = [],imgBox1,imgBox2,dragging = false,randomLeft = [0,this.canvasWidth];
    const loader = new PIXI.loaders.Loader();
    let resources = loader.resources;

    const rAF = window.requestAnimationFrame  ||
    window.webkitRequestAnimationFrame  ||
    window.mozRequestAnimationFrame   ||
    window.oRequestAnimationFrame   ||
    window.msRequestAnimationFrame    ||
    function (callback) { window.setTimeout(callback, 1000 / 60); };

    
    const loadImg = (e) =>{
      new Promise((resolve,reject)=>{
        if(e==0)loader.add(this.imgData[0]);
        if(e+1<this.imgData.length){
          loader.add(this.imgData[e+1])
        }
        loader.load().on("complete",function(){
          resolve();
        }).on("error",function(){
          reject();
        });
      }).then(()=>{
        //console.log('图片加载完毕');
        imgLoaded++;
        if(e===0)init();
      }).catch(()=>{
        //console.log('图片加载出错');
      });
    }

    loadImg(0);

    imgBox1 = new PIXI.Container();
    imgBox2 = new PIXI.Container();
    imgBox1.x = imgBox2.x = this.canvasWidth/2;
    imgBox1.y = imgBox2.y = this.canvasHeight/2;
    imgBox2.alpha = 0;

    stage.addChild(imgBox1,imgBox2);

    const init = ()=>{
      stage.interactive = true;
      stage.on('pointermove', (event)=>{
        if(dragging){
          if(this.scrollY)disX += (event.data.global.x - getX)/2;
          else if(Math.abs(event.data.global.x - getX)>Math.abs(getY - event.data.global.y))disX += (event.data.global.x - getX)/2;
          else disX += (getY - event.data.global.y)/2;
        }
        getX = event.data.global.x;
        getY = event.data.global.y;
      }).on('pointerdown', (event)=>{
        dragging = true;
        getX = event.data.global.x;
        getY = event.data.global.y;
      }).on('pointerup', dragEnd).on('pointerupoutside', dragEnd)

      for(let i=1;i<this.countNum;i++){
        randomLeft.push(Math.floor(this.canvasWidth/this.countNum*i));
      }
      randomLeft.sort((a,b)=>a-b);

      let _scale = Math.max(this.canvasWidth/resources[this.imgData[0]].data.width,this.canvasHeight/resources[this.imgData[0]].data.height);
      for(let i=0;i<this.countNum;i++){
        getKey[i] = 0;
        imgList1[i] = new Sprite(resources[this.imgData[0]].texture);
        imgList1[i].anchor.set(0.5); 
        imgList1[i].scale.set(_scale);
        imgList2[i] = new PIXI.Sprite(resources[this.imgData[0]].texture);
        imgList2[i].anchor.set(0.5); 
        imgList2[i].scale.set(_scale);

        maskList1[i] = new PIXI.Graphics();
        maskList1[i].beginFill(0x000000,0);
        maskList1[i].drawRect(0, 0, randomLeft[i+1]-randomLeft[i], this.canvasHeight);
        maskList1[i].x = randomLeft[i];
        maskList1[i].endFill();
        maskList2[i] = new PIXI.Graphics();
        maskList2[i].beginFill(0x000000,0);
        maskList2[i].drawRect(0, 0, randomLeft[i+1]-randomLeft[i], this.canvasHeight);
        maskList2[i].x = randomLeft[i];
        maskList2[i].endFill();


        imgList1[i].mask = maskList1[i];
        imgList2[i].mask = maskList2[i];
        imgBox1.addChild(imgList1[i]);
        imgBox2.addChild(imgList2[i]);
        stage.addChild(maskList1[i],maskList2[i]);
      }

      animate();

      if(this.wheel){
        const wheelStart = (event) => {
          if(event.deltaY>0)this.nextPage();
          else if(event.deltaY<0)this.prePage();
        }
        this.$el.addEventListener('wheel', wheelStart);
        this.$el.addEventListener('mousewheel', wheelStart);
        this.$el.addEventListener('DOMMouseScroll', wheelStart);
      }
    }

    const dragEnd = (auto) => {
      if((auto==2||disX>this.canvasWidth/1208*80)&&this.canChange&&this.activeIndex>0){
        dragging = true;
        this.canChange = false;
        this.activeIndex--;
        for(let i=0;i<this.countNum;i++){
          imgList2[i].texture = PIXI.Texture.fromImage(this.imgData[this.activeIndex]);
          TweenMax.to(imgList1[i], .6, {
            x: imgList1[i].x+randomLeft[1],
            ease: Power2.easeIn
          });
          TweenMax.fromTo(imgList2[i], .6,{x: randomLeft[i-1]-randomLeft[i]}, {
            x: 0,
            ease: Power2.easeIn
          });
          TweenMax.to(maskList1[i], .6, {
            x: randomLeft[i+1],
            width: 0,
            ease: Power2.easeIn
          });
          TweenMax.fromTo(maskList2[i], .6,{width:0}, {
            width: randomLeft[i+1]-randomLeft[i],
            ease: Power2.easeIn
          });
        }
        TweenMax.fromTo(imgBox2, .6,{alpha:0}, {
          alpha: 1,
          ease: Power2.easeIn,
          onComplete: ()=>{
            for(let i=0;i<this.countNum;i++){
              imgBox2.alpha = 0;
              getKey[i] = 0;
              maskList1[i].width = randomLeft[i+1]-randomLeft[i];
              maskList1[i].x = randomLeft[i];
              imgList1[i].texture = PIXI.Texture.fromImage(this.imgData[this.activeIndex]);
              imgList1[i].x = 0;
            }
            disX = 0;
            dragging = false
            this.canChange = true;
          }
        });
      }
      else if((auto==1||disX<this.canvasWidth/1208*-80)&&this.canChange&&this.activeIndex<this.imgData.length-1){
        dragging = true;
        this.canChange = false;
        this.activeIndex++;
        for(let i=0;i<this.countNum;i++){
          imgList2[i].texture = PIXI.Texture.fromImage(this.imgData[this.activeIndex]);
          TweenMax.to(imgList1[i], .6, {
            x: imgList1[i].x-randomLeft[1],
            ease: Power2.easeIn
          });
          TweenMax.fromTo(imgList2[i], .6,{x: randomLeft[i+1]-randomLeft[i]}, {
            x: 0,
            ease: Power2.easeIn
          });
          TweenMax.to(maskList1[i], .6, {
            width: 0,
            ease: Power2.easeIn
          });
          TweenMax.fromTo(maskList2[i], .6,{x: randomLeft[i+1],width:0}, {
            x: randomLeft[i],
            width: randomLeft[i+1]-randomLeft[i],
            ease: Power2.easeIn
          });
        }
        TweenMax.fromTo(imgBox2, .6,{alpha:0}, {
          alpha: 1,
          ease: Power2.easeIn,
          onComplete: ()=>{
            if(this.activeIndex==imgLoaded)loadImg(this.activeIndex);
            for(let i=0;i<this.countNum;i++){
              imgBox2.alpha = 0;
              getKey[i] = 0;
              maskList1[i].width = randomLeft[i+1]-randomLeft[i];
              maskList1[i].x = randomLeft[i];
              imgList1[i].texture = PIXI.Texture.fromImage(this.imgData[this.activeIndex]);
              imgList1[i].x = 0;
            }
            disX = 0;
            dragging = false;
            this.canChange = true;
          }
        });
      }
      else{
        dragging = false;
      }
    }
    this.dragEnd = dragEnd;

    const animate = () => {
      rAF(animate);

      if(this.canChange){
        if(!dragging){
          if(Math.abs(disX)>0.01)disX *= 0.8;
          else {
            disX = 0;
          }
        }
        if(disX>0){
          for(let i=0;i<this.countNum;i++){
            getKey[i] += (disX - getKey[i]) * ((this.countNum-i)*0.03+0.02);
            getKey[i] = Math.min(getKey[i],this.canvasWidth/2);
          }
        }
        else{
          for(let i=0;i<this.countNum;i++){
            getKey[i] += (disX - getKey[i]) * (i*0.03+0.02);
            getKey[i] = Math.max(getKey[i],this.canvasWidth/-2);
          }
        }

        for(let i=0;i<this.countNum;i++){
          if(getKey[i]>0){
            imgList1[i].x = getKey[i]*i/this.countNum;
            imgList1[i].alpha = Math.max(1-getKey[i]/this.canvasWidth*(this.countNum-i)/2,0.2);
          }
          else{
            imgList1[i].x = getKey[i]*(this.countNum-i-1)/this.countNum
            imgList1[i].alpha = Math.max(1+getKey[i]/this.canvasWidth*(i+1)/2,0.2);
          }
        }
      }
      renderer.render(stage);
    }

    this.$el.appendChild(renderer.view)
},
methods:{
  nextPage(){
    if(this.canChange&&this.activeIndex<this.imgData.length-1){
      this.dragEnd(1);
    }
  },
  prePage(){
    if(this.canChange&&this.activeIndex>0){
      this.dragEnd(2);
    }
  }
}
}
</script>
<style scoped>
  .shutterSlide{
    position: relative;
    canvas{ position: absolute; left: 0; top: 0; width: 100%; height: 100%}
  }
</style>
