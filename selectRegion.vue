<template>
	<view class="search-view">
		<uni-easyinput suffixIcon="down" v-model="selectedCityText" :placeholder="props.placeholder" @focus="openPopup()" @iconClick="openPopup()"></uni-easyinput>
		<uni-popup ref="popup" border-radius="10px 10px 0 0" type="bottom" background-color="#fff">
			<view class="popup-header">
				<text class="cancel" @click="cancel()">取消</text>
				<text class="title">{{ props.placeholder }}</text>
				<text class="confirm" @click="confirm()">确认</text>
			</view>
			<view class="popup-content">
				<view class="node-list">
					<view class="node-item" :class="{ selected: selectedNodeItemIndex == index }" v-for="(item, index) in nodeList" :key="index" @click="nodeClick(item, index)">
						{{ item.nodeText }}
					</view>
				</view>
				<view class="select-list">
					<view class="select-item" :class="{ selected: item.isSelected }" v-for="(item, index) in showData" :key="index" @click="itemClick(item, index)">
						<text>{{ item.text }}</text>
						<uni-icons v-if="!item.isSelected" class="icon" type="right" color="#bbb" size="20"></uni-icons>
						<uni-icons v-if="item.isSelected" class="selected-icon" type="checkmarkempty" color="#5ea94a" size="20"></uni-icons>
					</view>
				</view>
			</view>
		</uni-popup>
	</view>
</template>

<script setup>
import { ref, nextTick } from 'vue';
import { onReady } from '@dcloudio/uni-app';
import { getHttp } from '../../utils/request.js';

// 城市区划Pid
let cityPid = JSON.parse(uni.getStorageSync('UserData')).region.pid;

// 城市区划级联选择器数据
const cityData = ref([]);

// 选择的区划名称
const selectedRegionName = ref('');

// 城市区划级联选择器ref
const selectedCityDataPicker = ref();

const emits = defineEmits(['selected']);

const props = defineProps({
	placeholder: {
		type: String,
		default: '请选择所属区域'
	}
});

// 选择的城市
const selectedCity = ref('');

// 弹出层
const popup = ref();

// 节点列表
const nodeList = ref([
	{
		// 当前选择的节点索引，对应的是数据里面的索引
		nodeIndex: '',
		// 当前选择的节点层级，对应的是数据里第几层children
		nodeLevel: '',
		// 当前选择的节点名称
		nodeText: '请选择'
	}
]);

// 当前选择的节点列表里的节点索引，方便寻找目前展示的是哪个节点
const selectedNodeItemIndex = ref('0');

// 级联选择器展示数据
const showData = ref([]);

// 选择的城市文本
const selectedCityText = ref('');

/**
 * @description 获取城市区划数据
 * 此函数用于获取城市的区划数据，并按照层级结构组织这些数据。
 */
function getCityData() {
	uni.showLoading({
		title: '加载中...',
		mask: true
	});

	getHttp({
		url: '/query/region',
		data: { pid: cityPid }
	})
		.then((result) => {
			uni.hideLoading(); // 请求完成后隐藏加载框
			if (result.code !== 0) {
				// 请求失败，提示错误
				console.error('获取城市区划数据失败：', result);
				uni.showToast({
					title: result.msg,
					icon: 'none'
				});
				return;
			}

			if (cityData.value.length === 0) {
				// 如果城市数据为空
				// 使用map迭代填充初始城市数据
				result.data.map((item) => {
					cityData.value.push({
						text: item.name, // 节点显示名称
						value: item.id, // 节点唯一标识
						code: item.code, // 节点编码
						pid: item.pid, // 父节点唯一标识
						children: [], // 子节点集合
						isSelected: false, // 是否选中标志
						nodeLevel: 1 // 节点层级
					});
				});
				showData.value = cityData.value; // 更新展示数据
			} else {
				let currentArray = cityData.value; // 获取当前数组
				let isShowNewNode = false; // 标记是否需要显示新节点

				// 遍历节点列表
				nodeList.value.forEach((node) => {
					console.log('node', node);
					for (let level = 1; level <= node.nodeLevel; level++) {
						if (!currentArray.length || !Array.isArray(currentArray[node.nodeIndex])) {
							// 当前层级不存在子节点或不是数组类型
							if (result.data.length) {
								// 如果有结果数据
								result.data.forEach((item) => {
									// 查找匹配父节点
									if (currentArray[node.nodeIndex]?.value === item.pid) {
										const newNode = {
											text: item.name, // 节点显示名称
											value: item.id, // 节点唯一标识
											code: item.code, // 节点编码
											pid: item.pid, // 父节点唯一标识
											children: [], // 子节点集合
											isSelected: false, // 是否选中标志
											nodeLevel: node.nodeLevel + 1 // 新节点层级
										};
										currentArray[node.nodeIndex].children.push(newNode); // 添加新子节点
										showData.value = currentArray[node.nodeIndex].children; // 更新展示数据
										isShowNewNode = true; // 设置标记
									}
								});
							} else if (!result.data.length && currentArray[node.nodeIndex] && currentArray[node.nodeIndex].value === cityPid) {
								// 如果没有结果数据并且当前节点为指定PID，则删除子节点属性
								delete currentArray[node.nodeIndex].children;
							}
							if (currentArray[node.nodeIndex] && currentArray[node.nodeIndex].children) {
								currentArray = currentArray[node.nodeIndex].children; // 进入下一层级
							} else {
								return false;
							}
						}
					}
				});

				if (isShowNewNode) {
					// 如果需要显示新节点，则向节点列表中添加新节点信息
					nodeList.value.push({
						nodeIndex: '', // 节点索引
						nodeLevel: '', // 节点层级
						nodeText: '请选择' // 节点文本提示
					});
					selectedNodeItemIndex.value++;
				}
			}
		})
		.catch((err) => {
			// 捕获异常，输出错误
			console.error('获取城市区划数据失败：', err);
			uni.hideLoading();
		});
}

/**
 * 判断数组中是否存在具有指定属性值的对象
 *
 * @param {Array} arr - 需要检查的数组，预计每个元素是一个对象
 * @param {string} propertyName - 需要检查的对象属性名称
 * @param {*} propertyValue - 需要检查的对象属性值
 * @returns {boolean} - 如果数组中存在具有指定属性值的对象，则返回true，否则返回false
 */
function hasObjectWithProperty(arr, propertyName, propertyValue) {
	// 使用Array.prototype.some方法遍历数组中的每个对象，检查指定属性是否等于指定值
	return arr.some((obj) => obj[propertyName] === propertyValue);
}

/**
 * 打开弹窗
 */
function openPopup() {
	popup.value.open();
}

/**
 * 取消
 */
function cancel() {
	popup.value.close();
}

/**
 * 节点点击处理函数
 * @param {Object} item 点击的节点
 * @param {Number} index 点击节点的索引
 */
function nodeClick(item, index) {
	// 获取当前城市数据数组
	let currentArray = cityData.value;
	// 根据节点层级进行遍历
	for (let level = 1; level <= item.nodeLevel; level++) {
		if (level === item.nodeLevel) {
			// 如果是当前节点层级，设置展示数据和选中节点索引
			showData.value = currentArray;
			selectedNodeItemIndex.value = index;
		} else {
			// 否则，获取下一级的子数组
			currentArray = currentArray[nodeList.value[index - 1].nodeIndex].children;
		}
	}
	// 截取节点列表，保留到当前选中节点及之前的部分
	nodeList.value = nodeList.value.slice(0, selectedNodeItemIndex.value + 1);
}

/**
 * 列表项点击处理函数
 * @param {Object} selectedItem 选择的项
 * @param {Number} index 该项索引
 */
function itemClick(selectedItem, index) {
	// 设置选中的城市
	selectedCity.value = selectedItem;
	// 遍历展示数据，设置选中状态
	showData.value.forEach((item) => {
		if (item.value === selectedItem.value) {
			item.isSelected = true;
		} else {
			item.isSelected = false;
		}
	});
	// 设置城市父级 ID
	cityPid = selectedItem.value;
	// 更新节点列表中的当前节点信息
	nodeList.value[selectedNodeItemIndex.value] = {
		nodeIndex: index,
		nodeText: selectedItem.text,
		nodeLevel: selectedItem.nodeLevel
	};

	let viewData = [];
	nodeList.value.forEach((item) => {
		if (item.nodeText !== '请选择') {
			viewData.push(item.nodeText);
		}
	});

	selectedCityText.value = viewData.join('/');

	// 获取城市数据的函数
	getCityData();
}

/**
 * 确认处理函数
 */
function confirm() {
	// 触发名为'selected'的事件，并传递选中的城市值
	emits('selected', selectedCity.value);
	// 关闭弹出框（假设存在 popup 对象且有 close 方法）
	popup.value.close();
}

onReady(() => {
	getCityData();
});
</script>

<style lang="scss" scoped>
.popup-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 30rpx;
	border-bottom: 1px solid #f5f5f5;

	text {
		display: block;
		font-size: 18px;
	}

	.cancel {
		color: #888888;
	}

	.confirm {
		color: #5ea94a;
	}

	.title {
		font-size: 14px;
		font-weight: 600;
	}
}
.popup-content {
	height: 60vh;

	.node-list {
		padding: 20rpx 20rpx 0;
		display: flex;
		align-items: center;

		.node-item {
			padding: 20rpx 0;
			font-size: 14px;
			margin-right: 20px;

			&.selected {
				border-bottom: 1px solid #5ea94a;
			}
		}
	}

	.select-list {
		height: 90%;
		overflow: auto;

		.select-item {
			font-size: 14px;
			color: #3b4144;
			padding: 10px;
			display: flex;
			align-items: center;
			justify-content: space-between;
			border-bottom: 1px solid #eaecef;

			&.selected {
				color: #5ea94a;
			}
		}
	}
}
</style>
