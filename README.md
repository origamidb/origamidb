# origamidb

origami is an embeddable database

it aims to implement a novel distributed consensus model drawing from ideas used in CRDTs

it's built in layers of composed traits with the intent that porting to a new storage medium simply requires implementing a page store

the write path acts as a deterministic state machine with mutations flowing downwards through the layers. side effects are handled by returned intent, with the expectation that some outer network loop reconciles intent with side effect. as such, the core can be kept extremely portable to different runtime environments and machine architectures

most of the design and architecture goals currently exist in RCS messages and claude context
