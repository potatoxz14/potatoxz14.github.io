---
title: 🎉 Linux mkdir
summary: mkdir 源码分析
date: 2025-08-26

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
  - 源码详解
---


系统调用

```c
vfs_mkdir
  ->kernfs_iop_mkdir
    ->cgroup_mkdir
      ->cgroup_apply_control_enable
        ->css_create
          ->cpu_cgroup_css_alloc  //为此cgroup分配一个task_group
          ->online_css
            ->cpu_cgroup_css_online  //初始化此task_group
SYSCALL_DEFINE2(mkdir, const char __user *, pathname, umode_t, mode)
{
	return do_mkdirat(AT_FDCWD, getname(pathname), mode);
}
int do_mkdirat(int dfd, struct filename *name, umode_t mode)
{
	struct dentry *dentry;
	struct path path;
	int error;
	unsigned int lookup_flags = LOOKUP_DIRECTORY;

retry:
	dentry = filename_create(dfd, name, &path, lookup_flags); //为该目录创建dentry
	error = PTR_ERR(dentry);
	if (IS_ERR(dentry))
		goto out_putname;

	error = security_path_mkdir(&path, dentry,
			mode_strip_umask(path.dentry->d_inode, mode));
	if (!error) {
		struct user_namespace *mnt_userns;
		mnt_userns = mnt_user_ns(path.mnt);
		//调用对应文件系统的mkdir为该目录创建inode，对于ext4调用的是ext4_mkdir
        //cgroup是kernfs_iop_mkdir
		error = vfs_mkdir(mnt_userns, path.dentry->d_inode, dentry,
				  mode);
	}
	done_path_create(&path, dentry);
	if (retry_estale(error, lookup_flags)) {
		lookup_flags |= LOOKUP_REVAL;
		goto retry;
	}
out_putname:
	putname(name);
	return error;
}
int vfs_mkdir(struct user_namespace *mnt_userns, struct inode *dir,
	      struct dentry *dentry, umode_t mode)
{
	int error = may_create(mnt_userns, dir, dentry);
	unsigned max_links = dir->i_sb->s_max_links;

	if (error)
		return error;

	if (!dir->i_op->mkdir)
		return -EPERM;

	mode = vfs_prepare_mode(mnt_userns, dir, mode, S_IRWXUGO | S_ISVTX, 0);
	error = security_inode_mkdir(dir, dentry, mode);
	if (error)
		return error;

	if (max_links && dir->i_nlink >= max_links)
		return -EMLINK;

	error = dir->i_op->mkdir(mnt_userns, dir, dentry, mode);
	if (!error)
		fsnotify_mkdir(dir, dentry);
	return error;
}
static int kernfs_iop_mkdir(struct user_namespace *mnt_userns,
			    struct inode *dir, struct dentry *dentry,
			    umode_t mode)
{
	struct kernfs_node *parent = dir->i_private;
	struct kernfs_syscall_ops *scops = kernfs_root(parent)->syscall_ops;
	int ret;

	if (!scops || !scops->mkdir)
		return -EPERM;

	if (!kernfs_get_active(parent))
		return -ENODEV;

	ret = scops->mkdir(parent, dentry->d_name.name, mode);

	kernfs_put_active(parent);
	return ret;
}
```

do_rmdir

vfs_rmdir

```c
static int kernfs_iop_rmdir(struct inode *dir, struct dentry *dentry)
{
	struct kernfs_node *kn  = kernfs_dentry_node(dentry);
	struct kernfs_syscall_ops *scops = kernfs_root(kn)->syscall_ops;
	int ret;

	if (!scops || !scops->rmdir)
		return -EPERM;

	if (!kernfs_get_active(kn))
		return -ENODEV;

	ret = scops->rmdir(kn);
	
	kernfs_put_active(kn);
	return ret;
}
int vfs_rmdir(struct user_namespace *mnt_userns, struct inode *dir,
		     struct dentry *dentry)
{
	int error = may_delete(mnt_userns, dir, dentry, 1);

	if (error)
		return error;

	if (!dir->i_op->rmdir)
		return -EPERM;

	dget(dentry);
	inode_lock(dentry->d_inode);

	error = -EBUSY;
	if (is_local_mountpoint(dentry) ||
	    (dentry->d_inode->i_flags & S_KERNEL_FILE))
		goto out;

	error = security_inode_rmdir(dir, dentry);
	if (error)
		goto out;

	error = dir->i_op->rmdir(dir, dentry);
	if (error)
		goto out;
//-------------------------
	shrink_dcache_parent(dentry);
	dentry->d_inode->i_flags |= S_DEAD;
	dont_mount(dentry);
	detach_mounts(dentry);
    //外加一个d_delete_notify(dir, dentry); 是需要删除的东西，dir就是inode.
//-----------------------------
out:
	inode_unlock(dentry->d_inode);
	dput(dentry);
	if (!error)
		d_delete_notify(dir, dentry);  //这个地方释放了inode
	return error;
}
int do_rmdir(int dfd, struct filename *name)
{
	struct user_namespace *mnt_userns;
	int error;
	struct dentry *dentry;
	struct path path;
	struct qstr last;
	int type;
	unsigned int lookup_flags = 0;
retry:
	error = filename_parentat(dfd, name, lookup_flags, &path, &last, &type);
	if (error)
		goto exit1;

	switch (type) {
	case LAST_DOTDOT:
		error = -ENOTEMPTY;
		goto exit2;
	case LAST_DOT:
		error = -EINVAL;
		goto exit2;
	case LAST_ROOT:
		error = -EBUSY;
		goto exit2;
	}

	error = mnt_want_write(path.mnt);
	if (error)
		goto exit2;

	inode_lock_nested(path.dentry->d_inode, I_MUTEX_PARENT);
	dentry = __lookup_hash(&last, path.dentry, lookup_flags);
	error = PTR_ERR(dentry);
	if (IS_ERR(dentry))
		goto exit3;
	if (!dentry->d_inode) {
		error = -ENOENT;
		goto exit4;
	}
	error = security_path_rmdir(&path, dentry);
	if (error)
		goto exit4;
	mnt_userns = mnt_user_ns(path.mnt);
	error = vfs_rmdir(mnt_userns, path.dentry->d_inode, dentry);
exit4:
	dput(dentry);
exit3:
	inode_unlock(path.dentry->d_inode);
	mnt_drop_write(path.mnt);
exit2:
	path_put(&path);
	if (retry_estale(error, lookup_flags)) {
		lookup_flags |= LOOKUP_REVAL;
		goto retry;
	}
exit1:
	putname(name);
	return error;
}

SYSCALL_DEFINE1(rmdir, const char __user *, pathname)
{
	return do_rmdir(AT_FDCWD, getname(pathname));
}
struct inode *kernfs_get_inode(struct super_block *sb, struct kernfs_node *kn)
{
	struct inode *inode;

	inode = iget_locked(sb, kernfs_ino(kn));
	if (inode && (inode->i_state & I_NEW))
		kernfs_init_inode(kn, inode);

	return inode;
}
```

