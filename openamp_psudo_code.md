# openamp Psudo code:
The pseudo code illustrates the usage of the key remoteproc and RPMsg APIs from master and remote contexts.

Master:
---
```c
/* Example rmoteproc ops table - implementation is provided by the application */
struct remoteproc_ops rproc_ops = {
  .init = rproc_init,
  .remove = rproc_remove,
  .start = rproc_start,
  .stop = rproc_stop,
  .shutdown = rproc_shutdown,
  .mmap = rproc_mmap,
};

/* Example image store ops to fetch the remote image content from the memory and load
* them to destination memory - implementation is provided by the application.
*/
struct image_store_ops mem_image_store_ops = {
  .open = mem_image_open,
  .close = mem_image_close,
  .load = mem_image_load,
  .features = SUPPORT_SEEK,
};

/* Endpoint data receive call back function */
static int rpmsg_endpoint_cb(struct rpmsg_endpoint *ept, void *data, size_t len, uint32_t src, void *priv) {
  return RPMSG_SUCCESS;
}

/* Name service announcement call back function for channel creation */
static void rpmsg_name_service_bind_cb(struct rpmsg_device *rdev, const char *name, uint32_t dest) {
}

/* Name service announcement call back function for channel deletion */
static void rpmsg_service_unbind(struct rpmsg_endpoint *ept) {
}
int main(void) {
    struct rpmsg_device * rpmsg_dev;
    struct remoteproc rproc;
    void *store = RMT_IMAGE_MEM_ADDR;
    struct rpmsg_virtio_device *rpmsg_vdev;
    struct rpmsg_endpoint ept;
    struct virtio_device vdev;
    struct metal_io_region *shbuf_io;
    void *shbuf;
    metal_phys_addr_t pa;

    /* Initialize remoteproc instance */
    remoteproc_init(&rproc, rproc_ops, NULL);

    /* mmap shared memory */
    pa = SHARED_MEM_PA;
    (void *)remoteproc_mmap(&rproc , &pa, NULL, SHARED_MEM_SIZE,NORM_NSHARED_NCACHE|PRIV_RW_USER_RW, &shbuf_io);

    /* Configure remoteproc to get ready to load executable */
    remoteproc_config(&rproc, NULL);

    /* Load the image */
    remoteproc_load(&rproc, NULL, RMT_IMAGE_MEM_ADDR, &mem_image_store_ops,NULL);

    /* Start the remote processor */
    ret = remoteproc_start(&rproc);
    
    /* Setup the communication mechanism */
    vdev = remoteproc_create_virtio(&rproc, 0, VIRTIO_DEV_MASTER, NULL);

    /* Only rpmsg virtio master needs to initialize the shared buffers pool*/
     shbuf = metal_io_phys_to_virt(shbuf_io, SHARED_MEM_PA);
     rpmsg_virtio_init_shm_pool(&shpool, shbuf,(SHARED_MEM_SIZE - SHARED_BUF_OFFSET));

    /* Initialize the underlying virtio device */
    ret = rpmsg_init_vdev(rpmsg_vdev, vdev, rpmsg_name_service_bind_cb, shbuf_io, shpool);


    /* Get the rpmsg device */
    rpmsg_dev = rpmsg_virtio_get_rpmsg_device(rpmsg_vdev);
    :
    :
    /* Wait for the name service announcement before endpoint creation */
    wait();
    /* NS announcement is received, create the endpoint*/
    (void) rpmsg_create_ept(&ept, rdev, RPMSG_SERV_NAME, RPMSG_ADDR_ANY,  dest, rpmsg_endpoint_cb, rpmsg_service_unbind);


    /* Endpoint is created - send data using the rpmsg comm APIs */
    rpmsg_send(&ept, HELLO_MSG, strlen(HELLO_MSG));
    :
    :
     remoteproc_stop(rproc);
    remoteproc_shutdown(rproc);
}
```
---

Remote
---
```c
int main(void)
{
    void *rsc_table;
    int rsc_size;
    int ret;
    metal_phys_addr_t pa;
    struct rpmsg_device * rpmsg_dev;
     struct rpmsg_endpoint ept;
    struct remoteproc rproc;
    struct virtio_device vdev;
    struct metal_io_region *shbuf_io;
    void *shbuf;

    /* Get the resource table */
    rsc_table = get_resource_table(rsc_index, &rsc_size);

    /* Initialize remoteproc instance */
    remoteproc_init(&rproc, rproc_ops, NULL);

    /* mmap resource table */
    pa = (metal_phys_addr_t)rsc_table; 
    (void *)remoteproc_mmap(ret_rproc , &pa, NULL, rsc_size, NORM_NSHARED_NCACHE|PRIV_RW_USER_RW, &ret_rproc.rsc_io);

    /* mmap shared memory */
    pa = SHARED_MEM_PA;
    (void *)remoteproc_mmap(&rproc , &pa, NULL, SHARED_MEM_SIZE, NORM_NSHARED_NCACHE|PRIV_RW_USER_RW, &shbuf_io);

    /* Pass resource table to remoteproc for parsing */
    remoteproc_set_rsc_table(&rproc , rsc_table, rsc_size);

    /* Setup the virtio device for communication */
    vdev = remoteproc_create_virtio(&rproc , 0, VIRTIO_DEV_SLAVE, NULL);

    /* Initialize the underlying virtio device */
    ret = rpmsg_init_vdev(rpmsg_vdev, vdev, rpmsg_name_service_bind_cb, shbuf_io, NULL);

    /* Get the rpmsg device */
    rpmsg_dev = rpmsg_virtio_get_rpmsg_device(rpmsg_vdev);

    /* Create the endpoint */
    (void) rpmsg_create_ept(&ept, rdev, RPMSG_SERV_NAME, RPMSG_ADDR_ANY, RPMSG_ADDR_ANY, rpmsg_endpoint_cb, rpmsg_service_unbind);

    /* Wait for the endpoint to get ready */
    while(!is_rpmsg_ept_ready(&ept));

    /* Endpoint is created - send data using the rpmsg comm APIs */
    rpmsg_send(&ept, HELLO_MSG, strlen(HELLO_MSG));

    :
    :
}

```
---
