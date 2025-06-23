# DMA 專欄

## Mapping
### dma_map_page, dma_map_single
```
dma_map_single (dma_map_page)                               include\linux\dma-mapping.h
    └ dma_map_page_attrs                                           kernel\dma\mapping.c
        └ iommu_dma_map_page                                  drivers\iommu\dma-iommu.c
            └__iommu_dma_map
                ├ iommu_dma_alloc_iova
                │  └ alloc_iova_fast                               drivers\iommu\iova.c
                │       └ alloc_iova
                │           └ alloc_iova_mem
                │               └ kmem_cache_zalloc
                └ iommu_map                                       drivers\iommu\iommu.c
                   ├ iommu_sync_map
                   │    └ iommu_domain->ops->iotlb_sync_map
                   │        -> vendor driver
                   └ iommu_map_nosync
                        └ iommu_domain->ops->map_pages            include\linux\iommu.h
                            -> vendor driver
```

```c
static inline dma_addr_t dma_map_single_attrs(struct device *dev, void *ptr,
		size_t size, enum dma_data_direction dir, unsigned long attrs)
{
	/* DMA must never operate on areas that might be remapped. */
	if (dev_WARN_ONCE(dev, is_vmalloc_addr(ptr),
			  "rejecting DMA map of vmalloc memory\n"))
		return DMA_MAPPING_ERROR;
	debug_dma_map_single(dev, ptr, size);
	return dma_map_page_attrs(dev, virt_to_page(ptr), offset_in_page(ptr),
			size, dir, attrs);
}
```

```c
static inline dma_addr_t dma_map_page_attrs(struct device *dev,
		struct page *page, size_t offset, size_t size,
		enum dma_data_direction dir, unsigned long attrs)
{
	return DMA_MAPPING_ERROR;
}
```

### dma_map_sg
```
dma_map_sg                       include\linux\dma-mapping.h
    └ dma_map_sg_attrs                                           kernel\dma\mapping.c
        └ iommu_dma_map_sg                                  drivers\iommu\dma-iommu.c
            ├ for_each_sg { pci_p2pdma_bus_addr_map }
            ├ iommu_dma_alloc_iova
            │  └ alloc_iova_fast                               drivers\iommu\iova.c
            │       └ alloc_iova
            │           └ alloc_iova_mem
            │               └ kmem_cache_zalloc
            └ iommu_map_sg                                       drivers\iommu\iommu.c
               ├ sg_dma_is_bus_address
               ├ iommu_sync_map
               │    └ iommu_domain->ops->iotlb_sync_map
               │        -> vendor driver
               └ iommu_map_nosync
                    └ iommu_domain->ops->map_pages            include\linux\iommu.h
                        -> vendor driver
```

## Sync
### single
```
dma_sync_single_for_cpu                                     include\linux\dma-mapping.h
    └ __dma_sync_single_for_cpu                                    kernel\dma\mapping.c
        ├ dma_map_ops->sync_single_for_cpu
        └ iommu_dma_sync_single_for_cpu                       drivers\iommu\dma-iommu.c
            ├ dev_is_dma_coherent
            ├ iommu_iova_to_phys
            ├ arch_sync_dma_for_cpu
            └ swiotlb_sync_single_for_cpu
```
### sg


### For DMA Driver
```
dma_sync_sg_for_cpu                                  include\linux\dma-mapping.h
    └─> __dma_sync_sg_for_cpu                               kernel\dma\mapping.c
        └─> iommu_dma_sync_sg_for_cpu                  drivers\iommu\dma-iommu.c
            └─> iommu_dma_sync_single_for_cpu
                └─> swiotlb_sync_single_for_cpu          include/linux/swiotlb.h
```

## Appendix

How They Relate
* dma-mapping.h is the main API for drivers.
* dma-map-ops.h provides the abstraction for different backends (direct, IOMMU, etc.).
* dma-direct.h is one backend for direct mapping.
* dma-buf.h/dma-buf.c are for buffer sharing between devices.
* dma.c is legacy, for old-style DMA channels.
arch/arm64/mm/dma-mapping.c is the ARM64-specific implementation.
* IOMMU and DMA engine frameworks are additional, depending on platform and use case.

1. kernel/dma.c
Role:
Implements a legacy DMA channel allocator for ISA-style DMA channels (mostly for older hardware).
Key Functions:
request_dma() / free_dma(): Reserve and release DMA channels.
/proc/dma support: Exposes DMA channel usage to userspace via procfs.
Relevance:
Mostly relevant for old-style DMA (ISA, early PCI). Modern systems use more advanced APIs.
2. include/linux/dma-buf.h
Role:
Defines the DMA-BUF framework API, which allows sharing memory buffers between devices (e.g., GPU and display controller) without unnecessary copies.
Key Structures:
struct dma_buf: Represents a shared buffer.
struct dma_buf_ops: Exporter operations (attach, detach, map, unmap, etc.).
struct dma_buf_attachment: Tracks device-specific attachment to a buffer.
Key Functions:
dma_buf_export(), dma_buf_fd(), dma_buf_get(), dma_buf_put()
dma_buf_attach(), dma_buf_detach()
dma_buf_map_attachment(), dma_buf_unmap_attachment()
CPU access helpers: dma_buf_begin_cpu_access(), dma_buf_end_cpu_access()
Relevance:
Critical for modern buffer sharing between devices (especially in graphics/multimedia).
3. drivers/dma-buf/dma-buf.c
Role:
Implementation of the DMA-BUF framework described in dma-buf.h.
Key Points:
Manages buffer export/import, reference counting, attachment, mapping, and synchronization.
Handles file descriptor interface for userspace.
Implements locking, debugfs, and sysfs support for DMA-BUFs.
Relevance:
Critical for the actual mechanics of buffer sharing in the kernel.
4. include/linux/dma-direct.h
Role:
Defines the direct mapping implementation for DMA, i.e., when devices can access physical memory directly (no IOMMU translation).
Key Structures/Functions:
struct bus_dma_region: Describes address translation regions.
translate_phys_to_dma(), translate_dma_to_phys()
dma_direct_alloc(), dma_direct_free(), etc.
Relevance:
Important for understanding how physical addresses are mapped to device addresses on platforms without IOMMU.
5. include/linux/dma-map-ops.h
Role:
Defines the DMA mapping operations abstraction (struct dma_map_ops), which allows different architectures or platforms to provide their own DMA mapping implementations.
Key Structures/Functions:
struct dma_map_ops: Function pointers for alloc, free, map, unmap, sync, etc.
Helpers for setting/getting DMA ops for devices.
Support for CMA (Contiguous Memory Allocator), global pools, and more.
Relevance:
Critical for the abstraction of DMA operations across architectures.
6. include/linux/dma-mapping.h
Role:
Main public API for DMA mapping in the kernel.
What drivers use to allocate, map, and sync DMA buffers.
Key Functions:
dma_alloc_coherent(), dma_free_coherent()
dma_map_single(), dma_unmap_single()
dma_map_page(), dma_unmap_page()
dma_map_sg(), dma_unmap_sg()
Attribute and mask helpers, sync helpers, etc.
Relevance:
Critical for all kernel code that interacts with DMA.
7. arch/arm64/mm/dma-mapping.c
Role:
Architecture-specific implementation of DMA mapping for ARM64.
Key Functions:
arch_sync_dma_for_device(), arch_sync_dma_for_cpu()
arch_dma_prep_coherent(), arch_setup_dma_ops()
Relevance:
Important for understanding how DMA is handled on ARM64, but each architecture will have its own version.

8. Other Critical Files/Implementations
IOMMU-related files:
drivers/iommu/ and include/linux/iommu.h
If your platform uses an IOMMU, these files are critical for address translation and protection.
Architecture-specific DMA mapping:
arch/<arch>/mm/dma-mapping.c (like the ARM64 one you listed)
Each architecture may have its own implementation.
DMA Engine Framework:
drivers/dma/
For offloading memory copy/fill operations to dedicated DMA engines.
Documentation:
Documentation/core-api/dma-api.rst
Explains the DMA API, attributes, and best practices.