<template>
	<div class="article">
		<!-- 按钮区 -->
		<div class="btns">
			<el-select clearable style='width:120px' size='small' v-model="params.categoryId" placeholder="所有栏目">
		    <el-option v-for='c in categories' :label='c.name' :value='c.id' :key='c.id'></el-option>
		  </el-select>
		  <el-input
		  	style="width:130px"
		  	size="small"
			  placeholder="请输入关键字"
			  v-model="params.keywords"
			  clearable>
			</el-input>

			<el-button size='small' @click='toAddArticle'>发布资讯</el-button>
			<el-button size='small' @click='batchDeleteArticles'>批量删除</el-button>
		</div>
		<!-- 按钮区结束 -->
		<!-- 数据区 -->
		<!-- {{params}} -->
		<!-- {{multipleSelection}} -->
		<div class="article_tbl" v-loading='loading'>
			<el-table :border='true' size='small' :data="articles" style="width: 100%" @selection-change="handleSelectionChange">
				<el-table-column type="selection" width="55" fixed></el-table-column>
	      <el-table-column prop="title" label="文章标题"></el-table-column>
	      <el-table-column prop="category.name" label="所属栏目"  align="center" width="100"> </el-table-column>
	      <el-table-column prop="author" label="作者" width="100" align='center'> </el-table-column>
	       <el-table-column prop="publishtime" label="发布时间" width="180" align='center'></el-table-column>
	       <el-table-column prop="readtimes" label="阅读次数" width="90" align='center'></el-table-column>
	      <el-table-column label="操作" width='80' fixed="right" align='center'>
	      	<template slot-scope='{row}'>
	      		<i class="fa fa-trash" @click='deleteArticle(row.id)'></i>
	      		<i class="fa fa-pencil" @click='toUpdateArticle(row)'></i>
	      	</template>
	      </el-table-column>
	    </el-table>
		</div>
		<!-- 数据区结束 -->
		<!-- 分页 -->
		<div class="page"  v-if="total >= params.pageSize">
			<el-pagination
		    layout="prev, pager, next"
		    @current-change='handleCurrentChange'
		    :page-size='params.pageSize'
		    :total="total">
		  </el-pagination>
		</div>
		<!-- 分页结束 -->
		<!-- 模态框区 -->
		<el-dialog :before-close='handleBeforeClose' :close-on-press-escape='false' @close='handleCloseDialog' fullscreen :title="articleDialog.title" :visible.sync="articleDialog.visible">
		  <el-form label-position='right' ref='articleForm' :model="articleDialog.form" size='small' :rules="rules" >
		    <el-row>
		    	<el-col :span='18'>
						<el-form-item label="资讯标题" label-width="100px" prop="title">
				      <el-input v-model="articleDialog.form.title" autocomplete="off"></el-input>
				    </el-form-item>
		    	</el-col>
		    	<el-col :span='6'>
		    		<el-form-item label="所属栏目" label-width="100px" prop="categoryId">
				      <el-select v-model="articleDialog.form.categoryId" placeholder="一级栏目" style="width:100%">
				        <el-option :key='index' v-for='(c,index) in categories' :label="c.name" :value="c.id"></el-option>
				      </el-select>
				    </el-form-item>
		    	</el-col>
		    </el-row>
				<el-form-item label="资讯正文" label-width="100px">
					<mavon-editor style="min-height: 450px" ref=md v-model="articleDialog.form.content"/>
		    </el-form-item>
		  </el-form>
		  <div slot="footer" class="dialog-footer">
		    <!-- <el-button size='small' @click="closeArticleDialog">取 消</el-button> -->
		    <el-button size='small' type="primary" @click='saveOrUpdateArticle'>确认发布</el-button>
		  </div>
		</el-dialog>
		<!-- 模态框区结束 -->
	</div>
</template>
<script>
	import axios from '@/http/axios'
	export default {
		data(){
			return {
				loading:false,
				categories:[],
				articles:[],
				multipleSelection:[],
				total:0,
				params:{
					page:0,
					pageSize:10,
				},
				articleDialog:{
					title:'',
					visible:false,
					form:{
						liststyle:'style-one',
						fileIds:[]
					},
					fileList:[]
				},
				rules:{
					title:[{
						required: true, 
						message: '请输入标题',
						trigger: 'blur' 
					}],
					categoryId:[{
						required: true, 
						message: '请选择栏目',
						trigger: 'change' 
					}]
				}
			}
		},
		watch:{
			// 只要params中的任意参数发生改变，就要刷新页面
			params:{
				handler:function(){
					this.findAllArticles();
				},
				deep:true
			}
		},
		created(){
			this.findAllCategories();
			this.findAllArticles();
		},
		methods:{
			// 设置间歇自动保存
			startInterval(){
				
			},
			// 自动保存
			autoSave(){
				this.articleDialog.form.source = this.$refs.md.d_render;
				let url = '/manager/article/saveOrUpdateArticle'
				axios.post(url,this.articleDialog.form)
				.then(({data:result})=>{
					alert(JSON.stringify(result));
				});
			},
			
			toUpdateArticle(row){
				this.startInterval();
				// 1. 显示模态框
				this.articleDialog.title = '修改资讯';
				this.articleDialog.visible = true;
				// 2. 克隆当前行数据（避免错误修改）
				let article = _.cloneDeep(row);
				// 3. 处理附件默认显示
				this.articleDialog.fileList = article.articleFileVMs.map((item)=>{
					return {
						name:item.cmsFile.id,
						url:'http://134.175.154.93:8888/group1/'+item.cmsFile.id
					}
				})
				// 4. 处理表单数据		
				// 4.1 依赖栏目 category - > categoryId
				article.categoryId = article.category.id;
				delete article.category;
				// 4.2 依赖文件 articleFileVMs -> fileIds
				article.fileIds = article.articleFileVMs.map(item=>item.cmsFile.id);
				delete article.articleFileVMs;
				// 4.3 取消默认空值
				for(let key in article){
					let val = article[key]
					if(!val){
						delete article[key];
					}
				}
				//5 将处理后的结果与表单双向绑定
				this.articleDialog.form = article;
			},

			batchDeleteArticles(){
				let ids  = this.multipleSelection.map((item)=>{
					return item.id;
				})
				alert(ids);
			},
			handleSelectionChange(val){
				this.multipleSelection = val;
			},
			// 关闭模态框
			closeArticleDialog(){
				this.resetForm();
				this.articleDialog.visible = false;
			},
			// 模态框关闭之前的处理
			handleBeforeClose(done){
				this.$confirm('关闭模态框将放弃当前所有操作, 是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(()=>{
        	done();
        })
			},
			// 模态框关闭时候的回调函数
			handleCloseDialog(){
				// clearInterval(this.interval);
			},
			// 重置表单
			resetForm(){
				this.$refs.articleForm.resetFields();
			},
			// 保存或更新文章
			saveOrUpdateArticle(){
				this.$refs.articleForm.validate((valid)=>{
					if(valid){
						this.articleDialog.form.source = this.$refs.md.d_render;
						let url = '/manager/article/saveOrUpdateArticle'
						axios.post(url,this.articleDialog.form)
						.then(({data:result})=>{
							if(result.status = 200){
								//1. 关闭模态框
								this.closeArticleDialog();
								//2. 提示成功
								this.$notify({
				          title: '成功',
				          message: result.message,
				          type: 'success'
				        });
				        //3. 刷新
								this.findAllArticles();
							} else {
								this.$notify.error({
				          title: '错误',
				          message: result.message
				        });
							}
						})
						.catch((error)=>{
							this.$notify.error({title: '错误', message: '网络异常'});
						});
					}
				})
			},
			handlerUploadSuccess(response, file, fileList){
				file.name = response.data.id;
				this.articleDialog.form.fileIds.push(response.data.id);
			},
			// 删除附件
			handleUploadRemove(file){
				alert(1);
				//1. 调用接口删除图片
				axios.get('/manager/file/delete',{
					params:{
						id:file.name
					}
				})
				.then(()=>{
					this.$notify({title:'成功', message:'操作成功'})
					//2. 从fileIds中移除掉
					_.remove(this.articleDialog.form.fileIds,(id)=>{
						return id == file.name;
					});
					this.articleDialog.form.fileIds.push(1);
					this.articleDialog.form.fileIds.pop();
				})
				.catch(()=>{
					this.$notify.error({title:'错误', message:'网络异常'})
				})
			},
			toAddArticle(){
				this.startInterval();
				this.articleDialog.fileList = [];
				this.articleDialog.form = {
						liststyle:'style-one',
						fileIds:[]
				};
				this.articleDialog.title = '发布资讯';
				this.articleDialog.visible = true;
			},
			// 查询所有栏目
			findAllCategories(){
				let url = '/manager/category/findAllCategory';
				axios.get(url).then(({data:result})=>{
					this.categories = result.data;
				}).catch((error)=>{
					this.$notify.error({
						title:'错误',
						message:'网络异常'
					})
				})
			},
			handleCurrentChange(page){
				this.params.page = page-1;
			},
			// 查询所有文章
			findAllArticles(){
				this.loading = true;
				let url = '/manager/article/findArticle';
				axios.get(url,{
					params:this.params
				})
				.then(({data:result})=>{
					// 将查询到的数据绑定到模型中
					console.log(result);
					this.total = result.data.total;
					this.articles = result.data.list;
				}).catch((error)=>{
					this.$notify.error({
						title:'错误',
						message:'网络异常'
					})
				}).finally(()=>{
					this.loading = false;
				});
			},
			deleteArticle(id){
				this.$confirm('此操作将永久删除该数据, 是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        })
        .then(()=>{
        	let url = '/manager/article/deleteArticleById';
        	axios.get(url,{params:{id }})
        	.then(({data:result})=>{
        		this.$notify({
		          title: '成功',
		          message: result.message,
		          type: 'success'
		        });
		        this.findAllArticles();
        	})
        	.catch(()=>{
        		this.$notify.error({
		          title: '错误',
		          message: '删除异常'
		        });
        	});
        })
        .catch(()=>{});
			}
		}
	}
</script>

<style scoped>
	.list_style {

	}
	.list_style > div {
		width: 200px;
		height: 80px;
		overflow-y: hidden;
		border:1px solid #ededed;
		padding: .5em;
		border-radius: 5px;
	}
	.list_style > div.current{
		border-color: #419fff;
	}
	.list_style img {
		width: 100%;
	}
	.list_style > div.list_one {
		float: left;
	}
	.list_style > div.list_two {
		margin-left: 220px;
	}
</style>


