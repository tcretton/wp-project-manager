<template>
	<multiselect
		class="pm-common-multiselect"
		v-model="list"
        :options="lists"
        :multiple="multiple"
        :show-labels="true"
        :placeholder="options.placeholder"
        select-label=""
        selected-label="selected"
        deselect-label=""
        label="title"
        track-by="id"
        :allow-empty="allowEmpty"
        :loading="loadingListSearch"
        @search-change="asyncListsFind"
        @input="onChange"
    />

</template>

<script>
	export default {
		props: {
			allowEmpty: {
				type: [Boolean],
				default () {
					return false
				}
			},

			multiple: {
				type: [Boolean],
				default () {
					return false
				}
			},

            options: {
                type: [Object],
                default () {
                    return {
                        placeholder: __('Task Lists', 'wedevs-project-manager'),
                        projectId: false
                    }
                }
            },

            selectedLists: {
                type: [Object, Array, String],
                default () {
                    return ''
                }
            },

            projectId: {
                type: [Number, String, Boolean],
                default () {
                    return false
                }
            },

            withArchive: {
                type: [Boolean],
                default () {
                    return false
                }
            }
		},

		data () {
			return {
				//lists: [],
				timeout: '',
				loadingListSearch: false,
                listAbort: '',
                list: '',
                defaultProjectId: false
			}
		},

		components: {
			'multiselect': pm.Multiselect.Multiselect
		},

        computed: {
            lists () {
                var storeList = this.$store.state.taskCreateFormLists;
                
                const response = storeList.filter( list => parseInt(list.project_id) == parseInt(this.defaultProjectId) );

                return response;
            }
        },

		watch: {
            selectedLists () {
                this.formatSelectedListsId();
            },

            projectId () {
                this.setProjectId();
                this.setLists();
            }
        },

		created() {
            this.setProjectId();
			this.setLists();
		},

		methods: {
            // setSelectedList () {
            //     if(this.isEmpty(this.projectId)) {
            //         this.selectedLists =  '';
            //         return;
            //     }

            //     if(this.isEmpty(this.$store.state.taskCreateFormLists[this.projectId])) {
            //         this.selectedLists =  '';
            //         return;
            //     }
                
            //     this.selectedLists = this.$store.state.taskCreateFormLists[this.projectId].lists[0];
            // },

            setProjectId () {
                if(this.projectId) {
                    this.defaultProjectId = parseInt( this.projectId );
                } else if(this.options.projectId) {
                    this.defaultProjectId = parseInt( this.options.projectId );
                }
            },

            formatSelectedListsId () {
                var self = this;
                if( this.is_object(this.selectedLists) ) {
                    this.list = Object.assign({}, this.selectedLists)
                }
                
                if( this.is_array( this.selectedLists ) ) {
                    this.list = [];
                    this.selectedLists.forEach( (list) => {
                        list.id = parseInt( list.id );
                        self.list.push(list);
                    } )
                }
            },

            formatLists(lists) {
                var self = this;
                // self.lists = [];

                // lists.forEach(function(list) {
                //     let index = self.getIndex( self.lists, list.id, 'id' );

                //     if(index === false) {
                //         self.lists.push({
                //             id: list.id,
                //             title: list.title,
                //         });
                //     }
                // });
                // 
                
                if(this.isEmpty(lists)) {
                    return;
                }
                
                if(!this.is_array(lists)) {
                    lists = [lists];
                }
                
                self.$store.commit( 'updateTaskCreateFormLists', {
                    projectId: self.projectId,
                    lists: lists
                } )
                
                self.$emit('afterGetLists', self.lists);
            },

			onChange (val, el) {
				this.$emit('onChange', val);
			},

			setLists () {
				var lists = [],
					self = this;

				if ( self.lists.length ) {
                    self.$emit('afterGetLists', self.lists);
                }

				var args = {
					data: {
						project_id: this.defaultProjectId,
					},

					callback (res) {
                        self.$store.commit( 'updateTaskCreateFormLists', {
                            projectId: self.defaultProjectId,
                            lists: res.data
                        } )

                        //It will check new added list
                        self.$emit('afterRefreshLists', self.lists);
					}
				}

				this.getLists(args);
			},

			getLists (args) {
	            var self = this;
                var status = [1];

                if( self.withArchive ) {
                    status = [1,0];
                }

	            var request = {
	                url: self.base_url + 'pm/v2/projects/'+args.data.project_id+'/task-lists?with=incomplete_tasks,complete_tasks&incomplete_task_per_page=-1&complete_task_per_page=-1',
                    data: {
                        status: status
                    },
	                success (res) {
	                    if ( typeof args.callback != 'undefined' ) {
	                        args.callback (res);
	                    }
	                },
	                error (res) {

	                }
	            }

	            self.httpRequest(request);
	        },

	        asyncListsFind (title) {
                if(title == '') return;
                var self = this;
                clearTimeout(this.timeout);

                // Make a new timeout set to go off in 800ms
                this.timeout = setTimeout(function () {
                    self.findLists(title);
                }, 500);
            },

            findLists (title) {
                var self = this;

                if(self.listAbort) {
                    self.listAbort.abort();
                }

                var status = [1];

                if( self.withArchive ) {
                    status = [1,0];
                }

                var request = {
                    url: self.base_url + 'pm/v2/projects/'+this.defaultProjectId+'/task-lists?title='+title+'&with=incomplete_tasks,complete_tasks&incomplete_task_per_page=-1&complete_task_per_page=-1',
                    data: {
                        status: status
                    },
                    success: function(res) {
                        
                        self.loadingListSearch = false;
                        self.formatLists(res.data);
                    }
                }
                self.loadingListSearch = true;
                self.listAbort = self.httpRequest(request);
            }
		}
	}
</script>

<style lang="less">
    .pm-common-multiselect {
        min-height: auto;
        margin-right: 8px;

        .multiselect__single {
            margin-bottom: 0;
        }
        .multiselect__select {
            display: none;
        }
        .multiselect__input {
            border: none !important;
            box-shadow: none !important;
            margin: 0;
            font-size: 14px;
            vertical-align: baseline;
            height: 0;
        }
        .multiselect__element {
            .multiselect__option {
                font-weight: normal;
                white-space: normal;
                padding: 6px 12px;
                line-height: 25px;
                font-size: 14px;
                display: flex;

                .option-image-wrap {
                    .option__image {
                        border-radius: 100%;
                        height: 16px;
                        width: 16px;
                    }
                }
                .option__desc {
                    line-height: 20px;
                    font-size: 13px;
                    margin-left: 5px;
                }
            }

        }
        .multiselect__tags {
            min-height: auto;
            padding: 4px;
            border-color: #ddd;
            border-radius: 3px;
            white-space: normal;
            .multiselect__single {
                font-size: 12px;
            }
            .multiselect__tags-wrap {
                font-size: 12px;
            }

            .multiselect__spinner {
                position: absolute;
                right: 24px;
                top: 14px;
                width: auto;
                height: auto;
                z-index: 99;
            }

            .multiselect__tag {
                margin-bottom: 0;
                overflow: visible;
                border-radius: 3px;
                margin-top: 2px;
            }
        }
    }
</style>
