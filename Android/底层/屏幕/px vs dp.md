[Material Design关于px dp的讲解，非常好](https://m2.material.io/design/layout/pixel-density.html#pixel-density)。Pixel是像素，不同设备每英寸的pixel数量都不一样，有的是120，有的是160（pixel数/英寸），这个120或160称为屏幕的density，然后规定——在160density下，1dp == 1pixel大小。

那么density大于160，单位英寸下pixel数就高于160，但是因为要保证 **相同dp下物理长度相同**，所以density越高，1dp对应的pixel数越大，density越小，1dp对应的pixel数就越小，