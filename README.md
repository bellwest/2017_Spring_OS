# Project 04

# Geo-tagged File System

# Introduction
The objective of this project is to upgrade file system, geo-tagged ext2 file system. By tagged gps lcation, we also have to control the file permissions. If the distance between location of kernel and the file, which will be calculated from their gps coordinates, is shorter than sum of their accuracy, than our policy decides to give the user permission to access the file. By upgrading whole file system, we can understand how the kernel file system works and how the file system maintains whole the files in the system flawless.

# Implementation

## 1. `gps_location` definition
We made definition of `gps_location` in `include/linux/gps.h`.
```c
struct gps_location {
  int lat_integer;      /* latitude = lat_integer + lat_fractional * 10^(-6) */
  int lat_fractional;   /* range of latitude : [-90, 90] */
  int lng_integer;      /* longitude = lng_integer + lng_fractional * 10^(-6) */
  int lng_fractional;   /* range of longitude : [-180, 180] */
  int accuracy;         /* range of accuracy : nonnegative */
};
```
`lat` stands for latitude, and `lng` stands for longitude. Due to the kernel characteristic that the floating point variables cannot be used, each values are seperated by integer parts and fractional parts.

## 2. system call implementation

### 2-1. `sys_set_gps_location`
This system call is for updating the gps value of kernel. Prototype is below.
```c
int set_gps_location(struct gps_location __user *loc);
```
As you can see, the system call get's only a `gps_location` argument from user. By getting user space's data safely, we used `copy_from_user` and `access_ok` functions. And the kernel's `gps_location` is updated as same as the user's value. By protecting the kernel's location variable, we locked it by `write_lock`. Also this system call checks whether the user's `gps_location` value is appropriate or not.

### 2-2. `sys_get_gps_location`
This system call gets a file's path from user and give user back the corresponding file's `gps_location`. By find out corresponding `inode`, we used `user_path` function. 
```c
asmlinkage long sys_get_gps_location(const char __user *pathname, struct gps_location __user *loc)
{
  struct path fp;
  struct inode *inode;
  ...
  int err = user_path(pathname, &fp); /* get struct path from user */
  ...
  inode = fp.dentry->d_inode; /* get inode */
  ...
  {
    ...
    err = inode_permission(inode, MAY_READ);  /* checkout for permission */
    ...
  }
  ...
}
```
Also this system call calls the `inode_permission` like above, to checkout whether the file is okay to be read by user. If the system call decides that this file is forbidden to user, than `gps_location` of the file will not be returned.

## 3. `ext2_inode_operations` implementation
Like previous scheduler project, we have to implement three functions for `ext2` file system and link it to the `struct inode_operations`. There are several files which links the specific file system function pointers to global file system function, but we just linked only one file, `file.c`.

### 3-1. `ext2_set_gps_location`
This function simply set `gps_location` of the file same as kernel's. The important thing is that when does this function should be called. According to the project specification document, the `gps_location` of the file should be updated when the file is modified. So, we decided to track `i_mtime`, which is the member of `inode`, represents the modification time of the file. The reason is that when the file is modified, always `i_mtime` is modified together, and `gps_location` also should be modified. We checked all the kernel source and found out all the points where `i_mtime` is modified, and simply put the function close to them.

### 3-2. `ext2_get_gps_location`
This function is only called by `sys_get_gps_location`, which needs the `gps_location` of specific file. To get `gps_location` from `inode`, the function uses macro `EXT2_I`. `EXT2_I` gets `inode` and returns `ext2_inode_info` pointer. `ext2_inode_info` is the inode information which stays on the memory. 

### 3-3. `ext2_permission`
## 4. user space programs
how the permission is granted. 

# Lessons Learned
