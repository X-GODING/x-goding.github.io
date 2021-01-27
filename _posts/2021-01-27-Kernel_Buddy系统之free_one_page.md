>Kernel5.11.0-rc2

## 源码：

```c

static inline void __free_one_page(struct page *page,unsigned long pfn,struct zone *zone, unsigned int order,int migratetype, fpi_t fpi_flags)
{
	struct capture_control *capc = task_capc(zone);
	unsigned long buddy_pfn;
	unsigned long combined_pfn;
	unsigned int max_order;
	struct page *buddy;
	bool to_tail;

	max_order = min_t(unsigned int, MAX_ORDER - 1, pageblock_order); //-->(1)

	VM_BUG_ON(!zone_is_initialized(zone));
	VM_BUG_ON_PAGE(page->flags & PAGE_FLAGS_CHECK_AT_PREP, page);

	VM_BUG_ON(migratetype == -1);
	if (likely(!is_migrate_isolate(migratetype)))
		__mod_zone_freepage_state(zone, 1 << order, migratetype);

	VM_BUG_ON_PAGE(pfn & ((1 << order) - 1), page);
	VM_BUG_ON_PAGE(bad_range(zone, page), page);

continue_merging:
	while (order < max_order) {
		if (compaction_capture(capc, page, order, migratetype)) {
			__mod_zone_freepage_state(zone, -(1 << order),
								migratetype);
			return;
		}
		buddy_pfn = __find_buddy_pfn(pfn, order); //-->(2)
		buddy = page + (buddy_pfn - pfn); //-->(3)

		if (!pfn_valid_within(buddy_pfn))
			goto done_merging;
		if (!page_is_buddy(page, buddy, order)) //-->(4)
			goto done_merging;
		/*
		 * Our buddy is free or it is CONFIG_DEBUG_PAGEALLOC guard page,
		 * merge with it and move up one order.
		 */
		if (page_is_guard(buddy))
			clear_page_guard(zone, buddy, order, migratetype);
		else
			del_page_from_free_list(buddy, zone, order);
			
		combined_pfn = buddy_pfn & pfn; //-->(5)
		page = page + (combined_pfn - pfn);
		pfn = combined_pfn;
		order++;
	}
	if (order < MAX_ORDER - 1) {
		/* If we are here, it means order is >= pageblock_order.
		 * We want to prevent merge between freepages on isolate
		 * pageblock and normal pageblock. Without this, pageblock
		 * isolation could cause incorrect freepage or CMA accounting.
		 *
		 * We don't want to hit this code for the more frequent
		 * low-order merging.
		 */
		if (unlikely(has_isolate_pageblock(zone))) {
			int buddy_mt;

			buddy_pfn = __find_buddy_pfn(pfn, order);
			buddy = page + (buddy_pfn - pfn);
			buddy_mt = get_pageblock_migratetype(buddy);

			if (migratetype != buddy_mt
					&& (is_migrate_isolate(migratetype) ||
						is_migrate_isolate(buddy_mt)))
				goto done_merging;
		}
		max_order = order + 1;
		goto continue_merging;
	}

done_merging:
	set_buddy_order(page, order); //-->(6)

	if (fpi_flags & FPI_TO_TAIL)
		to_tail = true;
	else if (is_shuffle_order(order))
		to_tail = shuffle_pick_tail();
	else
		to_tail = buddy_merge_likely(pfn, buddy_pfn, page, order);

	if (to_tail) //-->(7)
		add_to_free_list_tail(page, zone, order, migratetype);
	else
		add_to_free_list(page, zone, order, migratetype);

	/* Notify page reporting subsystem of freed page */
	if (!(fpi_flags & FPI_SKIP_REPORT_NOTIFY))
		page_reporting_notify_free(order);//-->(8)
}
```

# (1) max_order
	min_t(unsigned int, MAX_ORDER - 1, pageblock_order);
	根据MAX_ORDER - 1 和 pageblock_order 获取其中系统支持的最小order，默认max_order=10

# （2）__find_buddy_pfn()
>根据物理地址page_pfn和order，获取page在order行上的伙伴的物理地址。
>可以这样理解，比如page的order=2，page_pfn=10,根据这个函数可以获取其在order=2列上的伙伴地址是10^(1<<2)=14
>后边就判断一下地址14是否存在，存在就合并（4个page和4个伙伴page合并就变成8个page块），不存在就退出循环


```c
/*
 * Locate the struct page for both the matching buddy in our
 * pair (buddy1) and the combined O(n+1) page they form (page).
 *
 * 1) Any buddy B1 will have an order O twin B2 which satisfies
 * the following equation:
 *     B2 = B1 ^ (1 << O)
 * For example, if the starting buddy (buddy2) is #8 its order
 * 1 buddy is #10:
 *     B2 = 8 ^ (1 << 1) = 8 ^ 2 = 10
 *
 * 2) Any buddy B will have an order O+1 parent P which
 * satisfies the following equation:
 *     P = B & ~(1 << O)
 *
 * Assumption: *_mem_map is contiguous at least up to MAX_ORDER
 */
static inline unsigned long
__find_buddy_pfn(unsigned long page_pfn, unsigned int order)
{
	return page_pfn ^ (1 << order);
}
```

# (3)
	计算page的伙伴页的首地址

# (4) page_is_buddy
	判断page的伙伴buddy是否存在


# (5)

```c
		combined_pfn = buddy_pfn & pfn; //-->(5)
		page = page + (combined_pfn - pfn);
		pfn = combined_pfn;
		order++;
```

	很多人解释是伙伴合并，当时一直不理解，就是算了一下地址哪来的合并操作啊，没看到有移动操作啊！
	后来理解了，其实就是从order到max_order循环的在每个order行上查找是否有它的伙伴，有的话计算出首地址，那么只要知道他在哪个order上就知道了有多少个page是连续的哈，这就是合并不需要移动。。。。


# (6) set_buddy_order

```c
static inline void set_buddy_order(struct page *page, unsigned int order)
{
	set_page_private(page, order);
	__SetPageBuddy(page);
}
```
	设置page的私有属性，伙伴系统通过这个值确定page属于哪个order


# (7)
	把该page添加到对应order的free_area队列中



# 整个过程

![在这里插入图片描述](https://img-blog.csdnimg.cn/202101271353296.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjk3ODY2Mg==,size_16,color_FFFFFF,t_70)
