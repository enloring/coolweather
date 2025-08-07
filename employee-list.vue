<template>
  <view class="page-container">
    <NavBarCustom bgColor="#FFFFFF" color="#1D2334" :border="false" title="员工库"
                  backPath="pages/workplatform/workplatform"
                  v-if="!isSearchActive">
      <template #right>
        <view class="search-icon" @click="activateSearch">
          <uni-icons type="search" size="22" color="#64748b"></uni-icons>
        </view>
      </template>
    </NavBarCustom>
    
    <view class="search-bar" v-if="isSearchActive">
      <view class="search-input-wrapper">
        <uni-icons type="search" size="18" color="#64748b" class="search-icon-input"></uni-icons>
        <input 
          class="search-input" 
          type="text" 
          confirm-type="search"
          v-model="searchKeyword"
          placeholder="搜索部门"
          focus
          @confirm="handleSearch"
          @input="handleSearchInput"
        />
        <uni-icons v-if="searchKeyword" type="clear" size="16" color="#94a3b8" @click="clearSearch" class="clear-icon"></uni-icons>
      </view>
      <view class="cancel-search" @click="cancelSearch">取消</view>
    </view>

    <!-- 搜索结果下拉框 -->
    <view class="search-results" v-if="isSearchActive && searchResults.length > 0">
      <scroll-view scroll-y class="search-results-list" :style="{ maxHeight: scrollViewHeight * 0.7 + 'px' }">
        <view 
          class="search-result-item" 
          v-for="(item, index) in searchResults" 
          :key="index"
          @click="goToDepartmentDetail(item)"
        >
          <view class="dept-tag" :style="{'background-color': getDeptTagColor(item.kind_code)}">
            {{ getDeptTagName(item.kind_code) }}
          </view>
          <view class="result-content">
            <view class="result-title">{{ item.dept_name }}</view>
            <view class="result-subtitle">{{ item.staff_info }}</view>
          </view>
        </view>
      </scroll-view>
    </view>

    <!-- 部门分类选择 -->
    <view class="category-selector" v-if="!isSearchActive || !searchResults.length">
      <view class="tabs-container">
        <view 
          v-for="(tab, index) in categoryTabs" 
          :key="index"
          class="category-option"
          :class="{ 'selected': currentTabIndex === index }"
          @click="switchTab(index)"
        >
          <view class="tab-name">{{ tab.name }}</view>
        </view>
      </view>
<!--      <view class="expand-button" @click="toggleCategoryExpand">-->
<!--        <view class="chevron" :class="{ 'chevron-up': isExpanded }"></view>-->
<!--      </view>-->
    </view>

    <!-- Swiper容器 -->
    <swiper 
      class="swiper-container" 
      :current="currentTabIndex" 
      @change="onSwiperChange"
      :style="{ height: scrollViewHeight + 'px' }"
      v-if="!isSearchActive || !searchResults.length"
    >
      <swiper-item v-for="(tab, tabIndex) in categoryTabs" :key="tabIndex">
        <!-- 部门列表 -->
        <scroll-view 
          scroll-y="true" 
          class="department-container" 
          @scroll="onScroll"
          :scroll-top="tabIndex === currentTabIndex ? scrollTop : 0"
          scroll-with-animation
          :show-scrollbar="false"
          @scrolltolower="onScrollToLower"
        >
          <!-- 部门列表项 -->
          <view class="department-tree">
            <view 
              class="department-tree-item" 
              v-for="(item, index) in getDeptTree(departmentLists[tabIndex] || [])" 
              :key="item.dept_id || index"
              @click="goToDepartmentDetail(item)"
            >
              <view class="tree-content" :class="{ 'has-parent': item.level > 0 }">
                <!-- 缩进线 -->
                <view class="tree-indent" v-if="item.level > 0">
                  <view class="indent-line"></view>
                </view>
                
                <!-- 部门内容 -->
                <view class="grid-content" :class="{ 
                  'child-dept': item.level > 0, 
                  'parent-dept': item.hasChildren,
                  'root-dept': item.level === 0
                }">
                  <view class="dept-header">
                    <view class="dept-badge" :style="{'background-color': getDeptColor(index)}">
                      {{ getFirstChar(item.dept_name) }}
                    </view>
                    <view class="dept-info">
                      <view class="dept-title">
                        <text v-if="item.level > 0" class="dept-path">{{ getDeptPath(item.dept_code, departmentLists[tabIndex] || []) }}</text>
                        <text class="dept-name">{{ item.dept_name }}</text>
                      </view>
                      <view class="dept-subtitle">{{ item.totalCount || item.dept_emp_count }} 人</view>
                    </view>
                  </view>
                  
                  <!-- 展开/收起按钮 -->
                  <view class="expand-button" v-if="item.hasChildren" @click.stop="toggleExpand(item.dept_id)">
                    <uni-icons :type="isExpanded(item.dept_id) ? 'down' : 'right'" size="18" color="#ffffff"></uni-icons>
                  </view>
                </view>
              </view>
            </view>
          </view>
          
          <!-- 加载状态 -->
          <view class="loading-indicator" v-if="isLoading && tabIndex === currentTabIndex">
            <view class="loading-spinner"></view>
            <text>正在加载</text>
          </view>
          
          <!-- 没有更多数据 -->
          <view class="end-of-list" v-if="!isLoading && !hasMoreData[tabIndex] && (departmentLists[tabIndex] || []).length > 0 && tabIndex === currentTabIndex">
            <view class="end-line"></view>
            <text>已加载全部</text>
            <view class="end-line"></view>
          </view>
          
          <!-- 空状态 -->
          <view class="empty-state" v-if="(departmentLists[tabIndex] || []).length === 0 && !isLoading && tabIndex === currentTabIndex">
            <view class="empty-icon"></view>
            <view class="empty-text">暂无部门数据</view>
          </view>
        </scroll-view>
      </swiper-item>
    </swiper>
  </view>
</template>

<script>
import { HrService } from '@/api/modules/hr/hr.js';
// #ifdef APP || APP-PLUS || H5
import NavBarCustom from '@/app_components/hr/app-navbar-custom/app-navbar-custom.vue';
// #endif

export default {
  name: 'AdminPositionList',
  components: {
    NavBarCustom
  },
  data() {
    return {
      departmentLists: [[], [], [], []], // 每个标签页对应的部门列表
      currentPages: [1, 1, 1, 1], // 每个标签页对应的当前页码
      pageSize: 100,
      pagesTotal: [1, 1, 1, 1], // 每个标签页对应的总页数
      isLoading: false,
      hasMoreData: [true, true, true, true], // 每个标签页是否有更多数据
      categoryTabs: [
        { name: '临床', value: '1' },
        { name: '医技', value: '2' },
        { name: '研究所', value: '3' },
        { name: '职能处室', value: '4' }
      ],
      currentTabIndex: 3, // 默认选中"职能科室"
      colorPalette: ['#4F46E5', '#2563EB', '#0891B2', '#0D9488', '#65A30D', '#CA8A04', '#DC2626', '#DB2777', '#7C3AED', '#C026D3'],
      scrollTop: 0,
      scrollViewHeight: 0,
      lastScrollLoad: 0,
      loadLock: false,
      // 搜索相关
      isSearchActive: false,
      searchKeyword: '',
      searchResults: [],
      searchTimeout: null,
      // 科室类型统计相关
      categoryStats: [[], [], [], []], // 每个分类的统计数据
      loadingStats: [false, false, false, false], // 每个分类的统计数据是否正在加载
      // 树形结构展开状态
      expandedDepts: new Set() // 记录已展开的部门ID
    }
  },
  onLoad() {
    this.calcScrollViewHeight();
    this.fetchDepartments();
  },
  
  onShow() {
    this.calcScrollViewHeight();
  },
  methods: {
    calcScrollViewHeight() {
      const windowHeight = uni.getSystemInfoSync().windowHeight;
      const navHeight = 50; // 导航栏高度估算
      const categoryHeight = 60; // 类别选择器高度估算
      const paddingHeight = 28; // 上下padding总和
      
      // 如果统计信息已展开，则减去统计信息的高度
      let statsHeight = 0;
      if (this.isExpanded) {
        statsHeight = 208; // 统计信息区域的高度（包括边距）
      }
      
      this.scrollViewHeight = windowHeight - navHeight - categoryHeight - paddingHeight - statsHeight;
    },
    
    getDeptColor(index) {
      return this.colorPalette[index % this.colorPalette.length];
    },
    
    getDeptTagColor(kindCode) {
      const index = parseInt(kindCode) || 0;
      return this.colorPalette[index % this.colorPalette.length];
    },
    
    getDeptTagName(kindCode) {
      const tab = this.categoryTabs.find(tab => tab.value === kindCode);
      return tab ? tab.name : '其他';
    },
    
    switchTab(index) {
      if (this.currentTabIndex === index) return;
      
      this.currentTabIndex = index;
      // 如果当前标签页的数据为空，则加载数据
      if (this.departmentLists[index].length === 0) {
        this.fetchDepartments();
      }
      
      // 如果统计区域已展开且当前标签页没有统计数据，则加载统计数据
      if (this.isExpanded && this.categoryStats[index].length === 0) {
        this.fetchCategoryStats();
      }
      
      // 重新计算滚动区域高度
      this.$nextTick(() => {
        this.calcScrollViewHeight();
      });
    },
    
    onSwiperChange(e) {
      const index = e.detail.current;
      this.switchTab(index);
    },
    
    // 切换分类展开状态
    toggleCategoryExpand() {
      this.isExpanded = !this.isExpanded;
      
      // 如果展开，则加载当前标签页的统计数据
      if (this.isExpanded) {
        // 如果当前标签页没有统计数据，则获取统计数据
        if (this.categoryStats[this.currentTabIndex].length === 0) {
          this.fetchCategoryStats();
        }
      }
      
      // 重新计算滚动区域高度
      this.$nextTick(() => {
        this.calcScrollViewHeight();
      });
    },
    
    // 获取科室类型统计数据
    async fetchCategoryStats() {
      if (this.loadingStats[this.currentTabIndex]) return;
      
      this.$set(this.loadingStats, this.currentTabIndex, true);
      
      try {
        const params = {
          kind_code: this.categoryTabs[this.currentTabIndex].value,
          page: 1,
          limit: 100 // 获取足够多的统计数据
        };
        
        const res = await HrService.queryDeptEmpList(params);
        
        if (res.data?.code === 1 && res.data.data) {
          this.$set(this.categoryStats, this.currentTabIndex, res.data.data|| []);
        } else {
          this.$set(this.categoryStats, this.currentTabIndex, []);
          uni.showToast({
            title: '获取统计数据失败',
            icon: 'none'
          });
        }
      } catch (error) {
        console.error('Error fetching category stats:', error);
        this.$set(this.categoryStats, this.currentTabIndex, []);
        uni.showToast({
          title: '获取统计数据失败',
          icon: 'none'
        });
      } finally {
        this.$set(this.loadingStats, this.currentTabIndex, false);
      }
    },
    
    resetList() {
      this.departmentLists[this.currentTabIndex] = [];
      this.currentPages[this.currentTabIndex] = 1;
      this.hasMoreData[this.currentTabIndex] = true;
      this.scrollTop = 0; // 重置滚动位置
      this.loadLock = false;
    },
    
    onScroll(e) {
      // 保存滚动位置，但不触发加载
    },
    
    onScrollToLower(e) {
      // 滚动到底部时加载更多
      if (this.hasMoreData[this.currentTabIndex] && !this.isLoading && !this.loadLock) {
        // 防止频繁触发
        const now = Date.now();
        if (now - this.lastScrollLoad < 800) return;
        
        this.lastScrollLoad = now;
        this.loadMore();
      }
    },
    
    loadMore() {
      if (this.hasMoreData[this.currentTabIndex] && !this.isLoading && !this.loadLock) {
        this.fetchDepartments();
      }
    },
    
    async fetchDepartments() {
      if (!this.hasMoreData[this.currentTabIndex] || this.isLoading) return;
      
      this.isLoading = true;
      this.loadLock = true; // 加载锁，防止重复触发
      
      try {
        const params = {
          kind_code: this.categoryTabs[this.currentTabIndex].value,
          page: this.currentPages[this.currentTabIndex],
          limit: this.pageSize
        };

        const res = await HrService.queryDeptEmpList(params);

        if (res.data?.code === 1 && res.data.data) {
          const newDepts = res.data.data.rows || [];
          this.pagesTotal[this.currentTabIndex] = res.data.data.pages || 1;
          
          this.hasMoreData[this.currentTabIndex] = this.currentPages[this.currentTabIndex] < this.pagesTotal[this.currentTabIndex];
          
          this.departmentLists[this.currentTabIndex] = [...this.departmentLists[this.currentTabIndex], ...newDepts];
          this.currentPages[this.currentTabIndex]++;
          
          // 自动展开dept_code小于等于6位的部门
          this.autoExpandDepartments(newDepts);
        } else {
          this.hasMoreData[this.currentTabIndex] = false;
          uni.showToast({
            title: '获取数据失败',
            icon: 'none'
          });
        }
      } catch (error) {
        console.error('Error fetching departments:', error);
        uni.showToast({
          title: '获取数据失败',
          icon: 'none'
        });
        this.hasMoreData[this.currentTabIndex] = false;
      } finally {
        this.isLoading = false;
        // 延时释放锁，防止快速连续触发
        setTimeout(() => {
          this.loadLock = false;
        }, 500);
      }
    },

    goToDepartmentDetail(dept) {
      uni.navigateTo({
        url: `/pages/apps/hr/base/dept-emp-detail?id=${dept.dept_id}&name=${encodeURIComponent(dept.dept_name)}`
      });
    },
    
    // 搜索相关方法
    activateSearch() {
      this.isSearchActive = true;
    },
    
    cancelSearch() {
      this.isSearchActive = false;
      this.searchKeyword = '';
      this.searchResults = [];
    },
    
    clearSearch() {
      this.searchKeyword = '';
      this.searchResults = [];
    },
    
    handleSearchInput() {
      // 防抖处理，避免频繁请求
      if (this.searchTimeout) {
        clearTimeout(this.searchTimeout);
      }
      
      if (this.searchKeyword.trim().length > 0) {
        this.searchTimeout = setTimeout(() => {
          this.performSearch();
        }, 300);
      } else {
        this.searchResults = [];
      }
    },
    
    handleSearch() {
      if (this.searchKeyword.trim().length > 0) {
        this.performSearch();
      }
    },
    
    async performSearch() {
      if (this.searchKeyword.trim().length === 0) return;
      
      this.isLoading = true;
      
      try {
        const params = {
          keyword: this.searchKeyword.trim()
        };
        
        const res = await HrService.queryDeptEmpList(params);
        
        if (res.data?.code === 1 && res.data.data) {
          this.searchResults = res.data.data.rows || [];
        } else {
          this.searchResults = [];
          uni.showToast({
            title: '搜索失败',
            icon: 'none'
          });
        }
      } catch (error) {
        console.error('Error searching departments:', error);
        uni.showToast({
          title: '搜索失败',
          icon: 'none'
        });
        this.searchResults = [];
      } finally {
        this.isLoading = false;
      }
    },

    onStatsSwiperChange(e) {
      const index = e.detail.current;
      this.switchTab(index);
    },

    parseAdminInfo(adminInfo) {
      if (!adminInfo) return [];
      
      // 解析格式为"职务名称(人数)，职务名称(人数)"的字符串
      const result = [];
      const items = adminInfo.split('，');
      
      items.forEach(item => {
        const match = item.match(/(.*)\((\d+)人\)/);
        if (match) {
          result.push({
            name: match[1],
            count: parseInt(match[2])
          });
        }
      });
      
      return result;
    },
    
    // 获取字符串中的第一个非数字字符
    getFirstChar(str) {
      if (!str) return '';
      
      // 匹配第一个非数字字符
      const match = str.match(/[^\d]/);
      if (match) {
        return match[0];
      }
      
      // 如果没有非数字字符，返回第一个字符
      return str.substring(0, 1);
    },
    
    // 基于dept_code构建部门树
    getDeptTree(departments) {
      if (!departments || departments.length === 0) return [];
      
      // 按dept_code排序
      const sortedDepts = departments.sort((a, b) => a.dept_code.localeCompare(b.dept_code));
      
      const result = [];
      
      sortedDepts.forEach(dept => {
        const level = this.getDeptLevel(dept);
        const hasChildren = this.hasChildrenByCode(dept.dept_id, departments);
        
        // 检查是否应该显示这个部门
        let shouldShow = true;
        
        // 检查所有父级部门是否都已展开
        if (level > 0) {
          shouldShow = this.isAllParentsExpanded(dept.super_code, departments);
        }
        
        if (shouldShow) {
          result.push({
            ...dept,
            level: level,
            hasChildren: hasChildren,
            totalCount: this.getDeptTotalCount(dept.dept_id, departments)
          });
        }
      });
      
      return result;
    },
    
    // 根据dept_level获取层级
    getDeptLevel(dept) {
      return dept.dept_level || 0;
    },
    
    // 根据super_code检查是否有子部门
    hasChildrenByCode(parentId, departments) {
      return departments.some(dept => dept.super_code === parentId);
    },
    
    // 检查部门是否已展开
    isExpanded(deptId) {
      return this.expandedDepts.has(deptId);
    },
    
    // 切换部门展开状态
    toggleExpand(deptId) {
      if (this.expandedDepts.has(deptId)) {
        this.expandedDepts.delete(deptId);
      } else {
        this.expandedDepts.add(deptId);
      }
    },
    
    // 检查所有父级部门是否都已展开
    isAllParentsExpanded(superCode, departments) {
      if (!superCode) return true;
      
      // 根据super_code查找父部门
      const parentDept = departments.find(d => d.dept_id === superCode);
      if (parentDept && !this.isExpanded(parentDept.dept_id)) {
        return false;
      }
      
      // 递归检查父部门的父部门
      if (parentDept && parentDept.super_code) {
        return this.isAllParentsExpanded(parentDept.super_code, departments);
      }
      
      return true;
    },
    
    // 获取部门路径
    getDeptPath(deptCode, departments) {
      if (!deptCode || deptCode.length <= 2) return '';
      
      const path = [];
      for (let i = 2; i < deptCode.length; i += 2) {
        const parentCode = deptCode.substring(0, i);
        const parentDept = departments.find(d => d.dept_code === parentCode);
        if (parentDept) {
          path.push(parentDept.dept_name);
        }
      }
      
      return path.join(' > ');
    },
    
    // 自动展开dept_level小于4的部门
    autoExpandDepartments(departments) {
      departments.forEach(dept => {
        if (dept.dept_level && dept.dept_level < 4) {
          this.expandedDepts.add(dept.dept_id);
        }
      });
    },
    
    // 获取指定部门的子部门
    getChildren(parentId, departments) {
      return departments.filter(dept => dept.super_code === parentId);
    },
    
    // 递归计算部门总人数（包括子部门）- 优化版本
    getDeptTotalCount(deptId, departments) {
      const currentDept = departments.find(d => d.dept_id === deptId);
      if (!currentDept) return 0;
      
      // 获取所有直接子部门
      const children = this.getChildren(deptId, departments);
      
      // 如果是叶子节点（没有子部门），直接返回当前部门人数
      if (children.length === 0) {
        return currentDept.dept_emp_count || 0;
      }
      
      // 如果是父节点，累加所有子部门的总人数（递归计算）
      let totalCount = 0;
      children.forEach(child => {
        totalCount += this.getDeptTotalCount(child.dept_id, departments);
      });
      
      // 父节点还要加上自己的直接员工数（如果有的话）
      totalCount += currentDept.dept_emp_count || 0;
      
      return totalCount;
    },
    
    // 检查是否有子部门（兼容旧方法）
    hasChildren(deptId, departments) {
      return departments.some(dept => dept.super_code === deptId);
    }
  }
};
</script>

<style>
.page-container {
  background-color: #f8fafc;
  display: flex;
  flex-direction: column;
  height: 100vh;
}

/* 搜索图标 */
.search-icon {
  padding: 8px 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* 搜索栏 */
.search-bar {
  padding: 8px 16px;
  background-color: #fff;
  display: flex;
  align-items: center;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  z-index: 10;
}

.search-input-wrapper {
  flex: 1;
  background-color: #f1f5f9;
  border-radius: 20px;
  padding: 8px 12px;
  display: flex;
  align-items: center;
}

.search-icon-input {
  margin-right: 8px;
}

.search-input {
  flex: 1;
  height: 20px;
  font-size: 14px;
  color: #1e293b;
  padding: 0;
  background: transparent;
}

.clear-icon {
  padding: 4px;
}

.cancel-search {
  padding: 0 12px;
  font-size: 14px;
  color: #3b82f6;
}

/* 搜索结果 */
.search-results {
  position: absolute;
  top: 58px;
  left: 0;
  right: 0;
  background-color: #fff;
  z-index: 9;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  border-bottom-left-radius: 12px;
  border-bottom-right-radius: 12px;
}

.search-results-list {
  max-height: 60vh;
  padding: 8px 0;
}

.search-result-item {
  padding: 12px 16px;
  display: flex;
  align-items: flex-start;
  border-bottom: 1px solid #f1f5f9;
}

.search-result-item:last-child {
  border-bottom: none;
}

.dept-tag {
  padding: 4px 8px;
  border-radius: 4px;
  color: #fff;
  font-size: 12px;
  margin-right: 12px;
  white-space: nowrap;
}

.result-content {
  flex: 1;
  overflow: hidden;
}

.result-title {
  font-size: 15px;
  font-weight: 500;
  color: #1e293b;
  margin-bottom: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.result-subtitle {
  font-size: 13px;
  color: #64748b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 分类选择器样式 */
.category-selector {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 10px;
  background-color: #fff;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  position: relative;
  z-index: 1;
}

.tabs-container {
  flex: 1;
  display: flex;
  justify-content: space-between;
}

.category-option {
  padding: 8px 12px;
  font-size: 14px;
  color: #64748b;
  border-radius: 0;
  transition: all 0.2s ease;
  position: relative;
  overflow: hidden;
  display: flex;
  align-items: center;
}

.tab-name {
  margin-right: 4px;
}

.category-option.selected {
  color: #3b82f6;
  font-weight: 500;
  background-color: transparent;
  box-shadow: none;
}

.category-option.selected:after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 20%;
  width: 60%;
  height: 2px;
  background-color: #3b82f6;
}

.expand-button {
  position: absolute;
  top: 8px;
  right: 8px;
  padding: 8px;
  background: linear-gradient(135deg, #3b82f6, #1d4ed8);
  border-radius: 50%;
  box-shadow: 0 2px 8px rgba(59, 130, 246, 0.3);
  border: 2px solid #ffffff;
  opacity: 1;
  transition: all 0.2s ease;
  z-index: 10;
  min-width: 32px;
  min-height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.expand-button:hover {
  background: linear-gradient(135deg, #1d4ed8, #1e40af);
  transform: scale(1.1);
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.4);
}

.expand-button:active {
  transform: scale(0.95);
}

.chevron {
  width: 8px;
  height: 8px;
  border-top: 1.5px solid #94a3b8;
  border-right: 1.5px solid #94a3b8;
  transform: rotate(135deg);
  margin-top: 3px;
}

.chevron-up {
  transform: rotate(-45deg);
  margin-top: 0;
}

/* 科室类型统计样式 */
.category-stats {
  background-color: #fff;
  margin-bottom: 8px;
  padding: 0;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  border-radius: 0 0 12px 12px;
  animation: slideDown 0.3s ease;
  overflow: hidden;
}

.stats-swiper {
  width: 100%;
  height: 200px;
}

@keyframes slideDown {
  from { opacity: 0; transform: translateY(-10px); }
  to { opacity: 1; transform: translateY(0); }
}

.stats-scroll-view {
  height: 100%;
}

.stats-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 16px;
}

.stats-summary {
  width: 100%;
  margin-bottom: 16px;
  text-align: center;
}

.stats-total {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 8px;
  background-color: #f1f5f9;
  padding: 8px 16px;
  border-radius: 20px;
  display: inline-flex;
}

.total-label {
  font-size: 14px;
  color: #64748b;
  margin-right: 4px;
}

.total-value {
  font-size: 16px;
  font-weight: 600;
  color: #3b82f6;
}

.stats-detail {
  width: 100%;
}

.detail-title {
  font-size: 14px;
  color: #64748b;
  margin-bottom: 12px;
  text-align: center;
  position: relative;
}

.detail-title:after {
  content: '';
  position: absolute;
  bottom: -6px;
  left: 50%;
  transform: translateX(-50%);
  width: 30px;
  height: 2px;
  background-color: #e2e8f0;
}

.detail-list {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 8px;
}

.detail-item {
  padding: 6px 10px;
  background-color: #f1f5f9;
  border-radius: 4px;
  font-size: 13px;
  color: #1e293b;
  display: flex;
  align-items: center;
}

.detail-name {
  margin-right: 4px;
}

.detail-count {
  color: #3b82f6;
  font-weight: 500;
}

.stats-loading, .stats-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 16px 0;
  height: 100%;
}

.stats-loading text, .stats-empty text {
  margin-top: 8px;
  font-size: 13px;
  color: #94a3b8;
}

/* Swiper容器 */
.swiper-container {
  flex: 1;
  width: 100%;
}

/* 部门列表容器 */
.department-container {
  flex: 1;
  background-color: #f8fafc;
  position: relative;
  z-index: 0;
  height: 100%;
}

.dept-title {
  font-size: 16px;
  font-weight: 600;
  color: #1e293b;
  margin-bottom: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.dept-subtitle {
  font-size: 13px;
  color: #64748b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 加载指示器 */
.loading-indicator {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 24px 0;
}

.loading-indicator text {
  margin-top: 12px;
  font-size: 14px;
  color: #94a3b8;
}

.loading-spinner {
  width: 24px;
  height: 24px;
  border: 3px solid rgba(59, 130, 246, 0.2);
  border-radius: 50%;
  border-top-color: #3b82f6;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* 列表结束指示器 */
.end-of-list {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px 32px;
}

.end-of-list text {
  margin: 0 12px;
  font-size: 12px;
  color: #94a3b8;
}

.end-line {
  flex: 1;
  height: 1px;
  background-color: #e2e8f0;
}

/* 空状态 */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 300px;
  padding: 32px;
}

.empty-icon {
  width: 80px;
  height: 80px;
  background-color: #f1f5f9;
  border-radius: 50%;
  margin-bottom: 16px;
  position: relative;
}

.empty-icon:before,
.empty-icon:after {
  content: '';
  position: absolute;
  background-color: #cbd5e1;
}

.empty-icon:before {
  width: 40px;
  height: 6px;
  border-radius: 3px;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.empty-icon:after {
  width: 6px;
  height: 40px;
  border-radius: 3px;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  opacity: 0.5;
}

.empty-text {
  font-size: 16px;
  color: #64748b;
  text-align: center;
}

/* 暗黑模式 */
@media (prefers-color-scheme: dark) {
  .page-container {
    background-color: #0f172a;
  }
  
  .category-selector {
    background-color: #1e293b;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
  }
  
  .category-option {
    color: #94a3b8;
  }
  
  .category-option.selected {
    color: #3b82f6;
    background-color: transparent;
  }
  
  .category-stats {
    background-color: #1e293b;
  }
  
  .stats-total {
    background-color: #334155;
  }
  
  .detail-title:after {
    background-color: #334155;
  }
  
  .detail-item {
    background-color: #334155;
    color: #e2e8f0;
  }
  
  .department-container {
    background-color: #0f172a;
  }
  
  .dept-title {
    color: #e2e8f0;
  }
  
  .dept-subtitle {
    color: #94a3b8;
  }
  
  .indent-line {
    background-color: #334155;
  }
  
  .indent-line::before {
    background-color: #334155;
  }
  
  .grid-content.child-dept {
    background: #1e293b;
    border-color: #334155;
  }
  
  .grid-content.parent-dept {
    background: #1e293b;
    border-color: #3b82f6;
    box-shadow: 0 2px 8px rgba(59, 130, 246, 0.2);
  }
}

.department-tree {
  padding: 12px 4px 0 4px;
}

.department-tree-item {
  margin-bottom: 8px;
}

.tree-content {
  display: flex;
  align-items: flex-start;
  position: relative;
}

.tree-content.has-parent {
  margin-left: 24px;
}

.tree-indent {
  position: absolute;
  left: -24px;
  top: 0;
  bottom: 0;
  width: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.indent-line {
  width: 2px;
  height: 100%;
  background-color: #e2e8f0;
  position: relative;
}

.indent-line::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 0;
  width: 16px;
  height: 2px;
  background-color: #e2e8f0;
}

.department-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 14px 0;
  padding: 12px 4px 0 4px;
  justify-content: flex-start;
}

.department-grid-item {
  width: 33.3333%;
  box-sizing: border-box;
  padding: 0 8px 8px 8px;
  display: flex;
  justify-content: center;
}
.grid-content {
  width: 100%;
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  display: flex;
  flex-direction: column;
  padding: 12px;
  transition: transform 0.15s;
  position: relative;
}

.dept-header {
  display: flex;
  align-items: center;
  width: 100%;
}

.dept-info {
  flex: 1;
  margin-left: 12px;
  overflow: hidden;
}

.grid-content.child-dept {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  margin-left: 8px;
}

.grid-content.parent-dept {
  background: #fff;
  border: 2px solid #3b82f6;
  box-shadow: 0 2px 8px rgba(59, 130, 246, 0.1);
}

.grid-content.root-dept {
  background: #fff;
  border: 3px solid #1e40af;
  box-shadow: 0 4px 12px rgba(30, 64, 175, 0.15);
}

.dept-path {
  font-size: 12px;
  color: #64748b;
  margin-bottom: 2px;
  display: block;
}

.dept-name {
  font-size: 15px;
  font-weight: 500;
  color: #1e293b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  width: 100%;
}

.hierarchy-indicators {
  position: absolute;
  top: 8px;
  right: 8px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.parent-indicator,
.child-indicator {
  opacity: 0.6;
  padding: 2px;
}
.grid-content:active {
  transform: scale(0.97);
}
.grid-content .dept-badge {
  width: 42px;
  height: 42px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  font-size: 18px;
  font-weight: 600;
  flex-shrink: 0;
}
.grid-content .dept-title {
  font-size: 15px;
  font-weight: 500;
  color: #1e293b;
  margin-bottom: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  width: 100%;
}
.grid-content .dept-subtitle {
  font-size: 12px;
  color: #64748b;
}
</style>