dispatch_group_t group = dispatch_group_create();
        dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
        dispatch_queue_t queue = dispatch_queue_create(NULL, DISPATCH_QUEUE_SERIAL);
        dispatch_group_async(group, queue, ^{
            for (int i = 0; i < self.dataArray.count; ++i) {
                WXOfficeModel *officeModel = self.dataArray[i];
                NSArray *departmentList = officeModel.departmentList;
                
                dispatch_group_t group1 = dispatch_group_create();
                dispatch_semaphore_t semaphore1 = dispatch_semaphore_create(0);
                dispatch_queue_t queue1 =dispatch_queue_create(NULL, DISPATCH_QUEUE_SERIAL);
                for (int i = 0; i < departmentList.count; ++i) {
                    WXDepartmentModel *departmentModel = departmentList[i];
                    dispatch_group_async(group1, queue1, ^{
                        [[WXMessageBusinessManeger sharedInstance] requestPeopleListForProject:[WXProject getlocalProject].pId office:officeModel.officeId depart:departmentModel.departmentId success:^(NSArray *array) {
                            NSMutableArray *peoplemModelArray = [NSMutableArray array];
                            for (int i = 0; i < array.count; ++i) {
                                NSDictionary *dict = array[i];
                                WXProjectUserData *userData = [WXProjectUserData modelWithDictionary:dict];
                                [peoplemModelArray addObject:userData];
                            }
                            departmentModel.memberArray = peoplemModelArray;
                            dispatch_semaphore_signal(semaphore1);
                        } failure:^(NSString *reason) {
                            dispatch_semaphore_signal(semaphore1);
                        }];
                        dispatch_semaphore_wait(semaphore1, DISPATCH_TIME_FOREVER);
                    });
                }
                dispatch_semaphore_t semaphore2 = dispatch_semaphore_create(0);
                dispatch_group_async(group1, queue1, ^{
                    [[WXMessageBusinessManeger sharedInstance] requestOrgLeadListForProject:[WXProject getlocalProject].pId office:officeModel.officeId success:^(NSArray *array) {
                        NSMutableArray *leadModelArray = [NSMutableArray array];
                        for (int i = 0; i < array.count; ++i) {
                            NSDictionary *dict = array[i];
                            WXProjectUserData *userData = [WXProjectUserData modelWithDictionary:dict];
                            [leadModelArray addObject:userData];
                        }
                        officeModel.principalArray = leadModelArray;
                        dispatch_semaphore_signal(semaphore2);
                    } failure:^(NSString *reason) {
                        dispatch_semaphore_signal(semaphore2);
                    }];
                    dispatch_semaphore_wait(semaphore2, DISPATCH_TIME_FOREVER);
                });
                
                dispatch_group_notify(group1, queue1, ^{
                    dispatch_semaphore_signal(semaphore);
                });
            }
            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        });
        dispatch_group_notify(group, queue, ^{
            [self initTableView];
        });
