# pylidc

`pylidc` is a python library intended to enhance workflow associated with the LIDC dataset, including utilities for both querying by attributes (e.g., collecting all annotations where malignancy is labeled as greater than 3 and spiculation is labeled a value equal to 1), and for common functional routines that act on the associated data (e.g., estimating the diameter or volume of a nodule from one of its annotations). Routines for visualizing the annotations, both atop the CT data and as a surface in 3D, are implemented. These functionalities are implemented via an object relational mapping (ORM), using `sqlalchemy` to an sqlite database containing the annotation information from the XML files provided by the LIDC dataset.

![](https://raw.githubusercontent.com/pylidc/pylidc/master/img/viz-in-scan-example.png)
![](https://raw.githubusercontent.com/pylidc/pylidc/master/img/viz-in-3d-example.png)

See below for installation details and for examples.

## Installation

`pylidc` has been tested on Linux on both python 2.7 and 3.5. It has been tested on MAC OSX with python 2.7. It has not been tested on Windows OS (although there is nothing in the library that is specific to any particular OS).

The package can be installed via `pip`:

    pip install pylidc

## Example usage

### Initial setup

The first thing you should do is tell the module where you store your dicom image files for LIDC dataset:

    >> import pylidc as pl
    >> pl.set_path_to_dicom_files('/path/to/big_external_drive/datasets/LIDC-IDRI')

The expected folder hierarchy in the specified path is the same as when you download the data from the TCIA download manager, i.e.,`PatientID` > `StudyInstanceUID` > `SeriesInstanceUID` > `*.dcm`. After you set this path initially, it will persist across sessions.

### Basic examples

There are three data models: `Scan`, `Annotation`, and `Contour`. The relationships are "one to many" for each model going left to right, i.e., `Scan`'s have many `Annotation`'s, and `Annotation`'s have many `Contour`'s. The main models to query are the `Scan` and `Annotation` models.

The main workhorse for querying is the `pylidc.query` function. This funciton just wraps the the `sqlalchemy.query` function. 

#### The `Scan` model

Here's some example usage for querying scan objects.

    import pylidc as pl
    
    qu = pl.query(pl.Scan).filter(pl.Scan.slice_thickness <= 1)
    print(qu.count())
    # => 97

    scan = qu.first()
    print(scan.patient_id, scan.pixel_spacing, scan.slice_thickness)
    # => LIDC-IDRI-0066, 0.63671875, 0.6

    print(len(scan.annotations))
    # => 11

    print(scan.get_path_to_dicom_files())
    # '/path/to/big_external_drive/datasets/LIDC-IDRI/LIDC-IDRI-0066/1.3.6.1.4.1.14519.5.2.1.6279.6001.143774983852765282237869625332/1.3.6.1.4.1.14519.5.2.1.6279.6001.430109407146633213496148200410'

You can engage an interactive slice view by calling:

    scan.visualize()

Note that calling `visualize` on a scan object doesn't include its annotation information -- you must call the `visualize_in_scan` member function of an `Annotation` object to do this.

#### The `Annotation` model

Let's grab the first annotation from the `Scan` object above:

    ann = scan.annotations[0]
    print(ann.scan.patient_id)
    # => LIDC-IDRI-0066

    print(ann.spiculation, ann.Spiculation())
    # => 3, Medium Spiculation

    print(ann.estimate_diameter(), ann.estimate_volume())
    # => 15.4920358194, 888.052284241

    ann.print_formatted_feature_table()
    # => Feature              Meaning                    # 
    # => -                    -                          - 
    # => Subtlety           | Obvious                  | 5 
    # => Internalstructure  | Soft Tissue              | 1 
    # => Calcification      | Absent                   | 6 
    # => Sphericity         | Ovoid                    | 3 
    # => Margin             | Poorly Defined           | 1 
    # => Lobulation         | Near Marked Lobulation   | 4 
    # => Spiculation        | Medium Spiculation       | 3 
    # => Texture            | Solid                    | 5 
    # => Malignancy         | Moderately Suspicious    | 4

    from pylidc.Annotation import feature_names as fnames
    fvals, fstrings = ann.feature_vals(return_str=True)
    print(fnames[0].title(), fstrings[0], fvals[0])
    # => Subtlety, Obvious, 5

Let's try a different query on the annotations directly:

    qu = pl.query(pl.Annotation).filter(pl.Annotation.lobulation > 3, pl.Annotation.malignancy == 5)
    print(qu.count())
    # => 183

    ann = qu.first()
    print(ann.lobulation, ann.Lobulation(), ann.malignancy, ann.Malignancy())
    # => 4, Near Marked Lobulation, 5, Highly Suspicious

    print(len(ann.contours))
    # => 8

    print(ann.contours_to_matrix().shape)
    # => (671, 3)

    print(ann.contours_to_matrix().mean(axis=0) - ann.centroid())
    # => [ 0.  0.  0.]

You can engage an interactive slice viewer that displays annotation values and the radiologist-drawn contours:

    ann.visualize_in_scan()

You can also view the nodule contours in 3d by calling:
    
    ann.visualize_in_3d()

### Advanced queries

#### Get a random result

One common objective for data exploration is to grab a random instance from some query. You can accomplish this by import `func` from `sqlalchemy`, and using `random`.

    from sqlalchemy import func
    scan = pl.query(pl.Scan).filter(pl.Scan.contrast_used == True).order_by(func.random()).first()
    ann  = pl.query(pl.Annotation).filter(pl.Annotation.malignancy == 5).order_by(func.random()).first()

The first query grabs a random `Scan` instance where contrast is used. The second query grabs a random `Annotation` instance where malignancy is equal to 5.

#### Query multiple model parameters with a join 

Another common objective is to query for an `Annotation` object which is constrained by its corresponding `Scan` in some way. For example:

    anns = pl.query(pl.Annotation).join(pl.Scan).filter(pl.Scan.slice_thickness < 1, pl.Annotation.malignancy != 3)

### Resampling the volumes

The `Annotation` member function, `uniform_cubic_resample`, takes a cubic region of interest with the centroid at the center of the volume. The corresponding CT value volume is resampled to have voxel spacing of 1 millimeter and a side length as given by the functions `side_length` parameter. Along with the uniformly resampled, cubic CT image volume, a corresponding boolean-valued volume is also returned that is 1 where the nodule exists in the resampled CT volume and 0 otherwise.
    
    ann = pl.query(pl.Annotation).first()
    vol, seg = ann.uniform_cubic_resample(side_length = 100)
    print(vol.shape, seg.shape)
    # => (101, 101, 101) (101, 101, 101)

    import matplotlib.pyplot as plt
    plt.imshow( vol[:,:,50] * (seg[:,:,50]*0.8 + 0.2), cmap=plt.cm.gray)
    plt.show()

![](https://raw.githubusercontent.com/pylidc/pylidc/master/img/resample-example.png)
