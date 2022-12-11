# GWO
Object-oriented programming in standard C

## Introduction
If you are like me and (1) do not like C++ as an OO language and (2) also have a lot of C code that would just take too much time to port to a proper OO language like Object Pascal and (3) that would still benefit from some OO features you have come to the right place. 
Let me illustrate the idea using examples from my Draughts Program GWD. I often have to generate moves, loop over the moves and sometimes have to print a move for debugging purposes. As you can imagine the code looks something like:

```
move_t moves[MOVES_MAX];
int nmoves;

nmoves = generate_moves(moves);

for (int imove = 0; imove < nmoves; imove++)
{
  int undo[MOVE_MAX];
  
  do_move(moves, imove, undo);
  ..
  if (bug)
  {
    char move_string[LINE_MAX];
    
    move2string(moves[imove], move_string);
    
    fprintf(stderr, "%s", move_string);
  }
  ..
  undo_move(moves, imove, undo);
}
```

Your code gets littered with local arrays like moves, undo and move_string that can also easily lead to errors if you nest calls, for example:

```
for (int imove = 0; imove < nmoves; imove++)
{
  int undo[MOVE_MAX];
  
  do_move(moves, imove, undo);
  ..
  if (bug)
  {
    char move_string[LINE_MAX];
    
    move2string(moves[imove], move_string);
    
    fprintf(stderr, "%s", move_string);
  }
  ..
  move_t moves2[MOVES_MAX];
  int nmoves2;

  nmoves2 = generate_moves(moves2);
  
  for (int jmove = 0; jmove < nmoves2; jmove++)
  {
    int undo2[MOVE_MAX];
  
    do_move(moves2, jmove, undo); //Oops should be undo2 here
    ..
    undo_move(moves2, jmove, undo); //Oops should be undo2 here
  }
  ..
  undo_move(moves, imove, undo);
}
```

As a  first improvement we can consolidate these arrays and  variables into a struct:

```
struct 
{
  move_t moves[MOVES_MAX];
  int nmoves;
  int undo[MOVES_MAX];
  char move_string[LINE_MAX];
} moves_t;

moves_t moves;

generate_moves(&moves); //this will now also set nmoves

for (int imove = 0; imove < moves.nmoves; imove++)
{
  do_move(&moves, imove, moves.undo);
  ..
  if (bug)
  {
    move2string(moves[imove], moves.move_string);
    
    fprintf(stderr, "%s", moves.move_string);
  }
  ..
  undo_move(&moves, imove, moves.undo);
}
```

Much better, but still does not prevent the undo2 error:

```
moves_t moves;

generate_moves(&moves); //this will now also set nmoves

for (int imove = 0; imove < moves.nmoves; imove++)
{
  do_move(&moves, imove, moves.undo);
  ..
  if (bug)
  {
    move2string(moves[imove], moves.move_string);
    
    fprintf(stderr, "%s", moves.move_string);
  }
  ..
  moves_t moves2;

  generate_moves(&moves2);
  
  for (int jmove = 0; jmove < moves2.nmoves; jmove++)
  { 
    do_move(&moves2, jmove, moves.undo); //Oops should be moves2.undo here
    ..
    undo_move(&moves2, jmove, moves.undo); //Oops should be moves2.undo here
  }
  ..
  undo_move(&moves, imove, moves.undo);
}
```

Let us now add OO-like methods to the moves_struct to get rid of these references to struct-local variables. The way to do this is to work with function pointers:

```
struct 
{
  move_t moves[MOVES_MAX];
  int nmoves;
  int undo[MOVES_MAX];
  char move_string[LINE_MAX];
  
  char * (*move2string)(struct moves *, int); //pointer to 'method'
} moves_t;
```

In the moves unit you define local functions that will use the struct-local variables and a 'constructor' for the moves_t 'object' that initializes the moves struct and links the function pointers to the 'methods':

```
//similar functions for gen_moves, do_move and undo_move
local char *moves_move2string(moves_t *self, int imove)
{
  snprintf(self->move_string, ..);
  return(self->move_string);
}

void create_moves(moves_t *self)
{
  self->nmoves = 0;
  //similar links to functions for gen_moves, do_move and undo_move
  self->move2string = moves_move2string;
}
```

OO languages will call the constructor for you, but now you have to do it yourself. Also you have to pass the 'self' object as an argument to the methods.
With this modification the code becomes:

```
moves_t moves;

create_moves(&moves); //call the constructor yourself

moves.gen_moves(&moves); //the function pointer has been set in the constructor

for (int imove = 0; imove < moves.nmoves; imove++)
{
  moves.do_move(&moves, imove); //you have to pass the 'self' object as an argument 
  ..
  if (bug)
  {
    fprintf(stderr, "%s", moves.move2string(&moves, imove));
  }
  ..
  moves_t moves2;

create_moves(&moves); //call the constructor yourself

  moves2.gen_moves(&moves2); //you can still have bugs like moves2->gen_moves(&moves);
  
  for (int jmove = 0; jmove < moves2.nmoves; jmove++)
  { 
    moves2.do_move(&moves2, jmove);
    ..
    moves2.undo_move(&moves2, jmove)
  }
  ..
  moves.undo_move(&moves, imove);
}
```

All in all much cleaner code to my opinion.

Let us generalize this approach now for the main 'objects' in my draughts program. These objects have a long life-time but I also have to loop over these objects, print the contents of the objects for debugging purposes etc. Again with a little discipline you can come a long way by introducing 'classes' that maintain lists of these 'objects'. The following is a worked-out example. All main objects in my draughts program derive from the generic class 'object':

```
//header
//generic class object

typedef void *(*ctor_t)(void); //generic constructor

typedef int (*iter_t)(void *); //generic iterator

typedef void (*pter_t)(void *); //generic printer

typedef struct class
{
  int (*construct_objects)(void *, struct class *);  //class constructor

  int nobjects_max;
  int nobjects;

  void **objects;

  ctor_t objects_ctor; //constructor
  iter_t objects_iter; //iterator

  PTHREAD_MUTEX_T objects_mutex; //mutex makes object creation and iteration thread-safe
} class_t;

//a class
//objects are derived from a class

//the class constructor construct_objects registers a created object
//in the class so that the class_iterator can loop over all objects and
//it returns the object_id.
//object_id can be used to create logical names for derived properties such as
//log-0.txt for the log-file of the first thread object
//log-1.txt for the log-file of the second thread object
//etc.
//you call the class constructor from the object constructor

local int construct_objects(void *self, class_t *class)
{
  PTHREAD_MUTEX_LOCK(class->objects_mutex);

  BUG(class->nobjects >= class->nobjects_max)

  int iobject = class->nobjects;

  //keep track of objects in class

  class->objects[iobject] = self;

  class->nobjects++;

  PTHREAD_MUTEX_UNLOCK(class->objects_mutex);

  return(iobject);
}

//you initialize a class by calling init_class
//ctor is a pointer to the constructor for objects of the class
//iter is a pointer to the iterator and is called when iterating over all objects of the class
//init_class registers the class constructor construct_objects

class_t *init_class(int nobjects_max, ctor_t ctor, iter_t iter)
{
  class_t *class;
 
  MALLOC(class, class_t, 1)

  //the class keeps track of the (number of) created objects

  class->nobjects_max = nobjects_max;

  class->nobjects = 0;

  MALLOC(class->objects, void *, nobjects_max)

  //register the class constructor

  class->construct_objects = construct_objects;

  //register the object constructor

  class->objects_ctor = ctor;

  //register the object iterator

  class->objects_iter = iter;

  //make the class constructor thread safe

  PTHREAD_MUTEX_INIT(class->objects_mutex);

  return(class);
}

//the object iterator can return an error value

void iterate_class(class_t *class)
{
  PTHREAD_MUTEX_LOCK(class->objects_mutex);

  BUG(class->objects_iter == NULL)

  int nerrors = 0;

  for (int iobject = 0; iobject < class->nobjects; iobject++)
    nerrors += class->objects_iter(class->objects[iobject]);
 
  BUG(nerrors > 0)
  
  PTHREAD_MUTEX_UNLOCK(class->objects_mutex);
}

//example

//objects are derived from the generic class object

local class_t *my_objects;

//define an object with properties and methods
//object_id is set by the class constructor

typedef struct
{
  //generic properties and methods

  int object_id;

  char object_stamp[LINE_MAX];

  iter_t iterate_object;

  //specific methods

  pter_t printf_object;

} my_object_t;

//the object printer

local void printf_my_object(void *self)
{
  my_object_t *my_object = (my_object_t *) self;

  PRINTF("printing my_object\n");

  PRINTF("object_stamp=%s\n", my_object->object_stamp);
  PRINTF("object_id=%d\n", my_object->object_id);
}

//the object constructor

local void *construct_my_object(void)
{
  my_object_t *self;
  
  MALLOC(self, my_object_t, 1)

  //call the class constructor

  self->object_id = my_objects->construct_objects(self, my_objects);

  //construct other parts of the object

  time_t t = time(NULL);
  (void) strftime(self->object_stamp, LINE_MAX,
    "%H:%M:%S-%d/%m/%Y", localtime(&t));

  //register object methods

  self->printf_object = printf_my_object;

  return(self);
}

//the object iterator

local int iterate_my_object(void *self)
{
  my_object_t *my_object = (my_object_t *) self;

  PRINTF("iterate object_id=%d\n", my_object->object_id);

  return(0);
}

void test_objects(void)
{
  //initialize the class 

  my_objects = init_class(1024, construct_my_object, iterate_my_object);

  my_object_t *a = my_objects->objects_ctor();

  a->printf_object(a);

  my_object_t *b = my_objects->objects_ctor();

  b->printf_object(b);

  iterate_class(my_objects);
}
```
