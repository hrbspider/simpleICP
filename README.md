# simpleICP

![simpleICP](data/dragon_iterations.png)

This repo contains implementations of a rather simple version of the [Iterative Closest Point (ICP) algorithm](https://en.wikipedia.org/wiki/Iterative_closest_point) in various languages.

Currently, an implementation is available for:

| Language | Code | Main dependencies |
| --- | --- | --- |
| C++ | [Link](c++) | [nanoflann](https://github.com/jlblancoc/nanoflann), [Eigen](http://eigen.tuxfamily.org), [cxxopts](https://github.com/jarro2783/cxxopts) |
| Julia | [Link](julia) | [NearestNeighbors.jl](https://github.com/KristofferC/NearestNeighbors.jl) |
| Matlab | [Link](matlab) | [Statistics and Machine Learning Toolbox](https://www.mathworks.com/products/statistics.html) |
| Octave | [Link](octave) | |
| Python | [Link](python) | [NumPy](https://numpy.org), [SciPy](https://scipy.org) |

I've tried to optimize the readability of the code, i.e. the code structure is as simple as possible and tests are rather rare.

The C++ version can be used through a cli interface.

![C/C++ CI](https://github.com/pglira/simpleICP/workflows/C/C++%20CI/badge.svg)

Also available at:

- Matlab: [![View simpleICP on File Exchange](https://www.mathworks.com/matlabcentral/images/matlab-file-exchange.svg)](https://www.mathworks.com/matlabcentral/fileexchange/81273-simpleicp)
- Python: [![](https://img.shields.io/pypi/v/simpleicp)](https://pypi.org/project/simpleicp)

## Features of the ICP algorithm

### Basic features

The following basic features are implemented in all languages:

- Usage of the signed **point-to-plane distance** (instead of the point-to-point distance) as error metric. Main reasons:
  - higher convergence speed, see e.g. [here](https://www.youtube.com/watch?v=LcghboLgTiA) and [here](https://ieeexplore.ieee.org/abstract/document/924423)
  - better final point cloud alignment (under the assumption that both point clouds are differently sampled, i.e. no real point-to-point correspondences exist)
- Estimation of a **rigid-body transformation** (rotation + translation) for the movable point cloud. The final transformation is given as homogeneous transformation matrix H:
  ```
  H = [R(0,0) R(0,1) R(0,2)   tx]
      [R(1,0) R(1,1) R(1,2)   ty]
      [R(2,0) R(2,1) R(2,2)   tz]
      [     0      0      0    1]
  ```
  where ``R`` is the rotation matrix and ``tx``, ``ty``, and ``tz`` are the components of the translation vector. Using ``H``, the movable point cloud can be transformed with:
  ```
  Xt = H*X
  ```
  where ``X`` is a 4-by-n matrix holding in each column the homogeneous coordinates ``x``, ``y``, ``z``, ``1`` of a single point, and ``Xt`` is the resulting 4-by-n matrix with the transformed points.
- Selection of a **fixed number of correspondences** between the fixed and the movable point cloud. Default is ``correspondences = 1000``.
- Automatic **rejection of potentially wrong correspondences** on the basis of
  1. the [median of absolute deviations](https://en.wikipedia.org/wiki/Median_absolute_deviation). A correspondence ``i`` is rejected if ``|dist_i-median(dists)| > 3*sig_mad``, where ``sig_mad = 1.4826*mad(dists)``.
  2. the planarity of the plane used to estimate the normal vector (see below). The planarity is defined as ``P = (ev2-ev3)/ev1`` (``ev1 >= ev2 >= ev3``), where ``ev`` are the eigenvalues of the covariance matrix of the points used to estimate the normal vector. A correspondence ``i`` is rejected if ``P_i < min_planarity``. Default is ``min_planarity = 0.3``.
- After each iteration a **convergence criteria** is tested: if the mean and the standard deviation of the point-to-plane distances do not change more than ``min_change`` percent, the iteration is stopped. Default is ``min_change = 1``.
- The normal vector of the plane (needed to compute the point-to-plane distance) is estimated from the fixed point cloud using a fixed number of neighbors. Default is ``neighbors = 10``.
- The point clouds must not fully overlap, i.e. a partial overlap of the point cloud is allowed. An example for such a case is the *Bunny* dataset, see [here](#test-data-sets). The initial overlapping area between two point  clouds can be defined by the parameter ``max_overlap_distance``. More specifically, the correspondences are only selected across points of the fixed point cloud for which the initial distance to the nearest neighbor of the movable point cloud is ``<= max_overlap_distance``.

## Output

All implementations generate the same screen output. This is an example from the C++ version for the *Bunny* dataset:

```
$ run_simpleicp.sh
Processing dataset "Bunny"
Create point cloud objects ...
Consider partial overlap of point clouds ...
Select points for correspondences within overlap area of fixed point cloud ...
Estimate normals of selected points ...
Start iterations ...
Iteration | correspondences | mean(residuals) |  std(residuals)
        0 |             961 |         -0.0321 |          0.2494
        1 |             961 |          0.0027 |          0.1349
        2 |             904 |         -0.0011 |          0.0540
        3 |             904 |         -0.0034 |          0.0390
        4 |             883 |         -0.0021 |          0.0325
        5 |             877 |         -0.0019 |          0.0284
        6 |             869 |         -0.0016 |          0.0243
        7 |             862 |         -0.0013 |          0.0213
        8 |             857 |         -0.0013 |          0.0191
        9 |             852 |         -0.0014 |          0.0188
       10 |             850 |         -0.0015 |          0.0186
Convergence criteria fulfilled -> stop iteration!
Estimated transformation matrix H:
[    0.990048    -0.172044     0.000977    -0.030468]
[    0.172035     0.990043    -0.001831    -0.022069]
[   -0.000834     0.001813     1.000022    -0.001465]
[    0.000000     0.000000     0.000000     1.000000]
Finished in 0.131 seconds!
```

## Test data sets

The test data sets are included in the [data](data) subfolder. An example call for each language can be found in the ``run_simpleicp.*`` files, e.g. [run_simpleicp.py](python/simpleicp/tests/run_simpleicp.py) for the python version.

| Dataset | | pc1 (no_pts) | pc2 (no_pts) | Overlap | Source |
| :--- | --- | --- | --- | --- | --- |
| *Dragon* | ![Dragon](/data/dragon_small.png) | [pc1](data/dragon1.xyz) (100k) | [pc2](data/dragon2.xyz) (100k) | full overlap | [The Stanford 3D Scanning Repository](http://graphics.stanford.edu/data/3Dscanrep/) |
| *Airborne Lidar* | ![AirborneLidar](/data/airborne_lidar_small.png) | [pc1](data/airborne_lidar1.xyz) (1340k) | [pc2](data/airborne_lidar2.xyz) (1340k) | full overlap | Airborne Lidar fligth campaign over Austrian Alps |
| *Terrestrial Lidar* | ![TerrestrialLidar](/data/terrestrial_lidar_small.png) | [pc1](data/terrestrial_lidar1.xyz) (1250k) | [pc2](data/terrestrial_lidar2.xyz) (1250k) | full overlap | Terrestrial Lidar point clouds of a stone block|
| *Bunny* | ![Bunny](/data/bunny_small.png) | [pc1](data/bunny_part1.xyz) (21k) | [pc2](data/bunny_part2.xyz) (22k) | partial overlap | [The Stanford 3D Scanning Repository](http://graphics.stanford.edu/data/3Dscanrep/) |

### Benchmark

These are the runtimes on my PC for the data sets above:

| Dataset             | C++   | Julia | Matlab | Octave* | Python |
| :--- | ---: | ---: | ---: | ---: | ---: |
| *Dragon*            | 0.16s | 3.99s |  1.34s | 95.7s   | 0.89s  |
| *Airborne Lidar*    | 3.98s | 5.38s | 15.08s | -       | 5.45s  |
| *Terrestrial Lidar* | 3.62s | 5.22s | 13.24s | -       | 5.68s  |
| *Bunny*             | 0.13s | 0.38s |  0.37s | 72.8s   | 0.80s  |

For all versions the same input parameters (``correspondences``, ``neighbors``, ...) are used.

**\*** Unfortunately, I haven't found an implementation of a kd tree in Octave (it is not yet implemented in the [Statistics](https://wiki.octave.org/Statistics_package) package). Thus, a (very time-consuming!) exhaustive nearest neighbor search is used instead. For larger datasets the Octave timings are missing, as the distance matrix does not fit into memory.

## References

Please cite related papers if you use this code:
```
@article{glira2015a,
  title={A Correspondence Framework for ALS Strip Adjustments based on Variants of the ICP Algorithm},
  author={Glira, Philipp and Pfeifer, Norbert and Briese, Christian and Ressl, Camillo},
  journal={Photogrammetrie-Fernerkundung-Geoinformation},
  volume={2015},
  number={4},
  pages={275--289},
  year={2015},
  publisher={E. Schweizerbart'sche Verlagsbuchhandlung}
}
```

## Related projects

- [globalICP](https://github.com/pglira/Point_cloud_tools_for_Matlab): A multi-scan ICP implementation for Matlab
