# Rocket File Cache
An in-memory file cache for the Rocket web framework.

Rocket File Cache can be used as a drop in replacement for Rocket's NamedFile when serving files.

This:
```rust
#[get("/<file..>")]
fn files(file: PathBuf) -> Option<NamedFile> {
    NamedFile::open(Path::new("static/").join(file)).ok()
}
```
Can be replaced with:
```rust
fn main() {
    let cache: Mutex<Cache> = Mutex::new(Cache::new(1024 * 1024 * 10)); // 10 megabytes
    rocket::ignite()
        .manage(cache)
        .mount("/", routes![files])
        .launch();
}

#[get("/<file..>")]
fn files(file: PathBuf, cache: State<Mutex<Cache>>) -> Option<CachedFile> {
    let pathbuf: PathBuf = Path::new("www/").join(file).to_owned();
    cache.lock().unwrap().get_or_cache(pathbuf)
}
```


# Should I use this?
Rocket File Cache keeps a set of frequently accessed files in memory so your webserver won't have to wait for your disk to read the files.
This should improve latency and throughput on systems that are bottlenecked on disk I/O.

# Performance
Running the bench tests that used to be in the repository on a computer with an Intel i7-6700K CPU @ 4.00GHz, with a Samsung NVME SSD 950 PRO 512 GB:
```
test tests::cache_access_10mib ... bench:   3,588,455 ns/iter (+/- 1,492,613)
test tests::cache_access_1mib  ... bench:      32,141 ns/iter (+/- 839)
test tests::cache_access_5mib  ... bench:   1,290,906 ns/iter (+/- 255,485)
test tests::clone5mib          ... bench:   1,276,393 ns/iter (+/- 37,072)
test tests::file_access_10mib  ... bench:   3,995,878 ns/iter (+/- 495,830)
test tests::file_access_1mib   ... bench:      77,488 ns/iter (+/- 973)
test tests::file_access_5mib   ... bench:   1,943,674 ns/iter (+/- 54,232)
```

There are across the board improvements for accessing the cache instead of the filesystem, with more significant gains made for smaller files, even with hardware that should not necessitate the use of this library.
That said, because the cache is guarded by a Mutex, synchronous access is impeded, possibly slowing down the effective serving rate of the webserver.

I have seen significant speedups for servers that serve small files that are only sporadically accessed.
I cannot recommend the use of this library outside of that use case until further benchmarks are performed.

# Warning
This crate is still under development.
Cache invalidation is still a work in progress.

# Documentation
Documentation can be found here: https://docs.rs/crate/rocket-file-cache