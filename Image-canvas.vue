<template>
	<div>
		<canvas id="imgcanvas" canvas-id="imgcanvas"
			:style="{width:wx_screen_width,height:wx_screen_height,backgroundColor:'black',position:'absolute'}"></canvas>
		<canvas id="imgcanvasmask" canvas-id="imgcanvasmask"
			:style="{width:wx_screen_width,height:wx_screen_height,position:'absolute'}"></canvas>
		<!-- #ifdef MP-WEIXIN || APP-PLUS  -->
		<div class="cropper-clip"
			:style="{width:cropperClip.style.width,height:cropperClip.style.height,left:cropperClip.style.left,top:cropperClip.style.top}">
			<div class="dot topleft" id="topleft" @touchend="touchend" @touchmove="touchmove"></div>
			<div class="dot topright" id="topright" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot topcenter" id="topcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot bottomleft" id="bottomleft" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot bottomcenter" id="bottomcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot bottomright" id="bottomright" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot leftcenter" id="leftcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot rightcenter" id="rightcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot center" id="center" @touchmove="touchmove" @touchend="touchend"></div>
		</div>
		<!-- #endif -->
		<!-- #ifdef H5 -->
		<div class="cropper-clip">
			<div class="dot topleft" id="topleft" @touchend="touchend" @touchmove="touchmove"></div>
			<div class="dot topright" id="topright" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot topcenter" id="topcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot bottomleft" id="bottomleft" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot bottomcenter" id="bottomcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot bottomright" id="bottomright" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot leftcenter" id="leftcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot rightcenter" id="rightcenter" @touchmove="touchmove" @touchend="touchend"></div>
			<div class="dot center" id="center" @touchmove="touchmove" @touchend="touchend"></div>
		</div>
		<!-- #endif -->
		<div class="toolbar">
			<button @tap="cropperClip_reset" class="btn-cancel">重置</button>
			<button @tap="cutDraw" class="btn-ok">确定</button>
		</div>
	</div>
</template>
<script>
let context;
let mask;
let min_cropbox_width = 30;
let min_cropbox_height = 30;
let image;
let stepX = 0
let stepY = 0
export default{
	data(){
		return {
			ImageCanvasPreview_show: false,
			Imagepreviewobj: {},
			// #ifdef MP-WEIXIN || APP-PLUS
			cropperClip: {
				style: {
					width: "0px",
					height: "0px",
					top: "0px",
					left: "0px"
				}
			},
			// #endif
			// #ifdef H5
			cropperClip: '',
			// #endif
			reReSizeDown: false,
			screen_height: '',
			screen_width: '',
			wx_screen_width: '',
			wx_screen_height: '',
		}
	},
	created(){
		let getWindowInfo = uni.getWindowInfo()
		this.screen_height = getWindowInfo.windowHeight; //可操作页面高度
		this.screen_width = getWindowInfo.windowWidth; //可操作页面宽度
		this.wx_screen_width = getWindowInfo.screenWidth + "px"
		this.wx_screen_height = getWindowInfo.windowHeight + "px"; //可操作页面高度
		if (this.condition.min_cropbox_width != undefined && this.condition.min_cropbox_height != undefined) {
			min_cropbox_width = this.condition.min_cropbox_width
			min_cropbox_height = this.condition.min_cropbox_height
		}
	},
	mounted(){
		// #ifdef H5
		this.cropperClip = document.querySelector(".cropper-clip");
		// #endif
		this.loadimg()
	},
	props:{
		Imageurl:{
			type: String,
			default: ''
		},
		condition:{
			type: Object,
			default(){
				return {}
			}
		}
	},
	methods:{
		touchmove(e) {
			let type;
			// #ifdef H5
			type = e.target.className.replace("dot ", "")
			// #endif
			// #ifdef MP-WEIXIN || APP-PLUS
			type = e.currentTarget.id
			// #endif
			this.reSizeDown(type, e.changedTouches[0])
		},
		touchend() {
			stepX = 0
			stepY = 0
		},
		reSizeDown(type, domEvent) {
			let that = this
			/* 根据缩放擦除蒙版 */
			function clearmask(x, y, w, h) {
				mask.fillStyle = "rgba(0,0,0,0.5)";
				mask.fillRect((that.screen_width / 2) - (image.width / 2), (that.screen_height / 2) - (image.height / 2), image.width,
					image.height);
				var posArr = [{
						x: x,
						y: y
					},
					{
						x: x + (w / 2),
						y: y
					},
					{
						x: x + w,
						y: y
					},
					{
						x: x,
						y: y + (h / 2)
					},
					{
						x: x + (w / 2),
						y: y + (h / 2)
					},
					{
						x: x + w,
						y: y + (h / 2)
					},
					{
						x: x,
						y: y + h
					},
					{
						x: x + (w / 2),
						y: y + h
					},
					{
						x: x + w,
						y: y + h
					}
				]
				mask.clearRect(x, y, w, h)
				posArr.map(e => {
					mask.beginPath();
					mask.arc(e.x, e.y, 5, 0, Math.PI * 2);
					mask.closePath()
					mask.fillStyle = '#1e9efb'
					mask.fill()
				})
				mask.draw()
			}
			this.reReSizeDown = true
			let distX;
			let disty;
			distX = domEvent.clientX - (Number(this.cropperClip.style.left.replace("px", "")))
			disty = domEvent.clientY - (Number(this.cropperClip.style.top.replace("px", "")))
			if (domEvent.clientX >= (this.screen_width / 2) - (image.width / 2) && domEvent.clientX < (this.screen_width / 2) - (image
					.width / 2) + image.width && domEvent.clientY >= (this.screen_height / 2) - (image.height / 2) && domEvent
				.clientY < (this.screen_height / 2) + (image.height / 2)) {
				if (type == "topleft") {
					let str_width = (Number(this.cropperClip.style.width.replace("px", ""))) - stepX
					if (str_width > min_cropbox_width) {
						stepX = distX - stepX
						this.cropperClip.style.width = str_width + "px"
						let str_1eft;
						str_1eft = (Number(this.cropperClip.style.left.replace("px", ""))) + distX - stepX
						this.cropperClip.style.left = str_1eft + "px"
					} else if (distX < 0) {
						stepX = distX - stepX
					}
					let str_height = (Number(this.cropperClip.style.height.replace("px", ""))) - stepY
					if (str_height > min_cropbox_height) {
						stepY = disty - stepY
						this.cropperClip.style.height = str_height + "px"
						let str_top = (Number(this.cropperClip.style.top.replace("px", ""))) + disty - stepY
						this.cropperClip.style.top = str_top + "px"
					} else if (disty < 0) {
						stepY = disty - stepY
					}
				} else if (type == "bottomleft") {
					let str_width = (Number(this.cropperClip.style.width.replace("px", ""))) - stepX
					if (str_width > min_cropbox_width) {
						stepX = distX - stepX
						this.cropperClip.style.width = str_width + "px"
						let str_1eft;
						str_1eft = (Number(this.cropperClip.style.left.replace("px", ""))) + distX - stepX
						this.cropperClip.style.left = str_1eft + "px"
					} else if (distX < 0) {
						stepX = distX - stepX
					}
					let str_height = disty
					if (str_height > min_cropbox_height) {
						this.cropperClip.style.height = str_height + "px"
					}
				} else if (type == "leftcenter") {
					let str_width = (Number(this.cropperClip.style.width.replace("px", ""))) - stepX
					if (str_width > min_cropbox_width) {
						stepX = distX - stepX
						this.cropperClip.style.width = str_width + "px"
						let str_1eft;
						str_1eft = (Number(this.cropperClip.style.left.replace("px", ""))) + distX - stepX
						this.cropperClip.style.left = str_1eft + "px"
					} else if (distX < 0) {
						stepX = distX - stepX
					}
				} else if (type == "topright") {
					let str_width = domEvent.clientX - (Number(this.cropperClip.style.left.replace("px", "")))
					if (str_width > min_cropbox_width) {
						this.cropperClip.style.width = str_width + "px"
					}
					let str_height = (Number(this.cropperClip.style.height.replace("px", ""))) - disty
					if (str_height > min_cropbox_height) {
						this.cropperClip.style.height = str_height + "px"
						let str_top = (Number(this.cropperClip.style.top.replace("px", ""))) + disty
						this.cropperClip.style.top = str_top + "px"
					}
				} else if (type == "topcenter") {
					let str_height = (Number(this.cropperClip.style.height.replace("px", ""))) - disty
					if (str_height > min_cropbox_height) {
						this.cropperClip.style.height = str_height + "px"
						let str_top = (Number(this.cropperClip.style.top.replace("px", ""))) + disty
						this.cropperClip.style.top = str_top + "px"
					}
				} else if (type == "bottomright") {
					let disty2 = domEvent.clientY - (Number(this.cropperClip.style.top.replace("px", "")))
					let str_width = domEvent.clientX - (Number(this.cropperClip.style.left.replace("px", "")))
					if (str_width > min_cropbox_width) {
						this.cropperClip.style.width = str_width + "px"
					}
					let str_height = disty2
					if (str_height > min_cropbox_height) {
						this.cropperClip.style.height = str_height + "px"
					}
				} else if (type == "rightcenter") {
					let str_width = domEvent.clientX - (Number(this.cropperClip.style.left.replace("px", "")))
					if (str_width > min_cropbox_width) {
						this.cropperClip.style.width = str_width + "px"
					}
				} else if (type == "bottomcenter") {
					let disty2 = domEvent.clientY - (Number(this.cropperClip.style.top.replace("px", "")));
					let str_height = disty2
					if (str_height > min_cropbox_height) {
						this.cropperClip.style.height = str_height + "px"
					}
				} else if (type == "center") {
					let str_top = (Number(this.cropperClip.style.top.replace("px", ""))) + disty - ((Number(this.cropperClip.style.height
						.replace("px", ""))) / 2)
					if (str_top - 1 <= (this.screen_height / 2) - (image.height / 2)) {
						str_top = (this.screen_height / 2) - (image.height / 2)
					} else if (str_top + 2 >= ((this.screen_height / 2) - (image.height / 2)) + image.height - (Number(this.cropperClip
							.style.height.replace("px", "")))) {
						str_top = ((this.screen_height / 2) - (image.height / 2)) + image.height - (Number(this.cropperClip.style.height
							.replace("px", "")))
					}
					this.cropperClip.style.top = str_top + "px"
					stepX = distX - stepX
					let str_1eft;
					str_1eft = (Number(this.cropperClip.style.left.replace("px", ""))) + distX - ((Number(this.cropperClip.style.width
						.replace("px", ""))) / 2)
					if (str_1eft - 1 <= (this.screen_width / 2) - (image.width / 2))
						str_1eft = (this.screen_width / 2) - (image.width / 2)
					else if (str_1eft + 2 >= ((this.screen_width / 2) - (image.width / 2)) + image.width - (Number(this.cropperClip.style
							.width.replace("px", ""))))
						str_1eft = ((this.screen_width / 2) - (image.width / 2)) + image.width - (Number(this.cropperClip.style.width
							.replace("px", "")))
					this.cropperClip.style.left = str_1eft + "px"

				}
				let left_mask = parseInt(this.cropperClip.style.left)
				let top_mask = parseInt(this.cropperClip.style.top)
				let width_mask = parseInt(this.cropperClip.style.width) + 2
				let height_mask = parseInt(this.cropperClip.style.height) + 2
				clearmask(left_mask, top_mask, width_mask, height_mask)
			}
		},
		/* 加载图片 */
		async loadimg() {
			let that = this
			// #ifdef H5
			context = uni.createCanvasContext('imgcanvas')
			mask = uni.createCanvasContext('imgcanvasmask')
			// #endif
			// #ifdef MP-WEIXIN || APP-PLUS
			context = uni.createCanvasContext('imgcanvas', this)
			mask = uni.createCanvasContext('imgcanvasmask', this)
			// #endif
			uni.showLoading({
				mask: true,
				title: '图片加载中'
			})

			uni.getImageInfo({
				src: this.Imageurl,
				success: function(image1) {
					console.log('图片加载完毕')
					uni.hideLoading()
					image = image1

					imageloadover()

				}
			});
			/* 启用蒙板 */
			function drawModal() {
				mask.fillStyle = "rgba(0,0,0,0.5)";
				mask.fillRect((that.screen_width / 2) - (image.width / 2), (that.screen_height / 2) - (image.height / 2), image
					.width, image.height);
				mask.draw()
			};
			function imageloadover() {
				//缩放图片
				let width_1 = that.screen_width * 0.9
				let height_1 = width_1 / (image.width / image.height)

				if (height_1 > that.screen_height * 0.6) {
					width_1 = (width_1 / height_1) * that.screen_height * 0.6
					height_1 = that.screen_height * 0.6
				}
				image.width = width_1
				image.height = height_1
				context.drawImage(image.path, (that.screen_width / 2) - (image.width / 2), (that.screen_height / 2) - (image.height /
					2), image.width, image.height);
				context.draw()
				/* 初始化裁剪框 */
				that.cropperClip_reset()
			}
		},
		/* 裁剪框重置事件 */ //之所以减1是因为边框的宽也要算进去
		cropperClip_reset() {
			this.cropperClip.style.width = `${image.width}px`;
			this.cropperClip.style.height = `${image.height}px`;
			this.cropperClip.style.left = `${((this.screen_width / 2) - (image.width / 2))-1}px`;
			this.cropperClip.style.top = `${((this.screen_height / 2) - (image.height / 2))-1}px`;
			mask.fillStyle = "rgba(0,0,0,0.5)";
			mask.fillRect((this.screen_width / 2) - (image.width / 2), (this.screen_height / 2) - (image.height / 2), image.width,
				image.height);
			var x = (this.screen_width / 2) - (image.width / 2);
			var y = (this.screen_height / 2) - (image.height / 2);
			var w = image.width;
			var h = image.height;
			var posArr = [{
					x: x,
					y: y
				},
				{
					x: x + (w / 2),
					y: y
				},
				{
					x: x + w,
					y: y
				},
				{
					x: x,
					y: y + (h / 2)
				},
				{
					x: x + (w / 2),
					y: y + (h / 2)
				},
				{
					x: x + w,
					y: y + (h / 2)
				},
				{
					x: x,
					y: y + h
				},
				{
					x: x + (w / 2),
					y: y + h
				},
				{
					x: x + w,
					y: y + h
				}
			]
			mask.clearRect((Number(this.cropperClip.style.left.replace("px", ""))), (Number(this.cropperClip.style.top.replace("px",
				""))), (Number(this.cropperClip.style.width.replace("px", ""))), (Number(this.cropperClip.style.height.replace(
				"px", ""))))
				
			posArr.map(e => {
				mask.beginPath();
				mask.arc(e.x, e.y, 5, 0, Math.PI * 2);
				mask.closePath()
				mask.fillStyle = '#1e9efb'
				mask.fill()
			})
			mask.draw()
		},

		/* 裁剪事件 */
		cutDraw() {
			if (this.condition.min_result_imgwidth != undefined && this.condition.min_result_imgheight != undefined) {
				if ((Number(this.cropperClip.style.width.replace("px", ""))) < this.condition.min_result_imgwidth || (Number(
						this.cropperClip.style.height.replace("px", ""))) < this.condition.min_result_imgheight) {
					this.$emit('cutwidthorheightnotenough', true)
					return;
				}
			}
			let tempheight = image.width / (Number(this.cropperClip.style.width.replace("px", "")) / Number(this.cropperClip.style.height
				.replace("px", "")))
			uni.canvasToTempFilePath({
					x: Number(this.cropperClip.style.left.replace("px", "")),
					y: Number(this.cropperClip.style.top.replace("px", "")),
					width: Number(this.cropperClip.style.width.replace("px", "")),
					height: Number(this.cropperClip.style.height.replace("px", "")),
					destWidth: Number(this.cropperClip.style.width.replace("px", "")),
					destHeight: Number(this.cropperClip.style.height.replace("px", "")),
					canvasId: 'imgcanvas',
					success: (res) => {
						this.Imagepreviewobj = {
							src: res.tempFilePath,
							width: image.width,
							height: image.height
						}
						this.ImageCanvasPreview_show = true
					},
					fail: (error) => {
						console.log("请换张后台允许跨域的图片")
						console.log(error)
					}
				}
				// #ifdef MP-WEIXIN || APP-PLUS
				, this
				// #endif
			)
		}
	},
}
</script>
<style scoped lang="scss">
	.toolbar {
		position: absolute;
		width: 100%;
		height: 100upx;
		left: 0upx;
		bottom: 0upx;
		box-sizing: border-box;
		border-bottom: 1px solid #C0C0C0;
		background: #F8F8F8;
	}
	.btn-cancel {
		position: absolute;
		left: 100upx;
		top: 12upx;
		font-size: 30upx;
		line-height: 30upx;
		padding: 20upx;
		color: #333333;
	}
	.btn-ok {
		position: absolute;
		right: 100upx;
		top: 12upx;
		font-size: 30upx;
		line-height: 30upx;
		padding: 20upx;
		color: #333333;
	}
	#imgcanvas {
		position: absolute;
		width: 100%;
		height: 100%;
		background-color: black;
	}

	#imgcanvasmask {
		position: absolute;
		width: 100%;
		height: 100%;
	}

	.Image-canvas-navigationbar {
		position: absolute;
		width: 100%;
		height: 12%;
		background-color: white;
		display: flex;
		justify-content: flex-start;
		align-items: center;

		text {
			transform: translateY(50%);
		}


	}

	.cropper-clip {
		position: absolute;
		cursor: all-scroll;
		border: 1px solid rgb(30, 158, 251);
	}

	.dot {
		width: 30px;
		height: 30px;

		/*  #ifdef APP-PLUS */
		width: 30px;
		height: 30px;
		/* #endif */
		border-radius: 50%;
		// background: #1e9efb;
		position: absolute;

	}

	.topleft {
		top: -15px;
		left: -15px;
		cursor: nwse-resize;
	}

	.topright {
		top: -15px;
		right: -15px;
		cursor: nesw-resize;
	}

	.topcenter {
		top: -15px;
		left: 50%;
		transform: translateX(-50%);
		cursor: ns-resize;
	}

	.bottomcenter {
		bottom: -15px;
		left: 50%;
		transform: translateX(-50%);
		cursor: ns-resize;
	}

	.bottomleft {
		bottom: -15px;
		left: -15px;
		cursor: nesw-resize;
	}

	.bottomright {
		bottom: -15px;
		right: -15px;
		cursor: nwse-resize;
	}

	.leftcenter {
		top: 50%;
		transform: translateY(-50%);
		left: -15px;
		cursor: ew-resize;
	}

	.rightcenter {
		top: 50%;
		transform: translateY(-50%);
		right: -15px;
		cursor: ew-resize;
	}

	.imgcanvascutbut {
		position: absolute;
		left: 50%;
		transform: translateX(-50%);
		bottom: 10%;
		width: 90%;
		height: 100rpx;
		background-color: #1e9efb;
		border-radius: 40rpx;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.imgcanvasresetbut {
		position: absolute;
		left: 5%;
		bottom: 10%;
		width: 180rpx;
		height: 100rpx;
		border-radius: 40rpx;
		display: flex;
		justify-content: center;
		align-items: center;
		background-color: gainsboro;
	}

	.center {
		position: absolute;
		left: 50%;
		top: 50%;
		transform: translate(-50%, -50%);
		opacity: 0.6;
		border-radius: 50%;

		// box-shadow: 0 0 0 10rpx rgba(2, 238, 255, 0.2);
	}
</style>