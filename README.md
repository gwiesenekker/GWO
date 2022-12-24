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

//in OO style you could also use malloc() to create the object but remember you have to free it later
/*
moves_t *create_moves(void)
{
  moves_t *self;
  
  self = (moves_t *) MALLOC(sizeof(moves_t));
  self->nmoves = 0;
  self->moves2string = moves_moves2string;
  return(self);
}
..
moves_t *moves = create_moves();
..
free(moves);
*/
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

  create_moves(&moves2); //call the constructor yourself

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

//a generic constructor. The generic constructor is registered wih the class object
//so that you can initialize an object using the syntax class_object->ctor(),
//but this is more a matter of taste, you could also call the constructor directly as in the example above.
//if the object constructor has arguments you cannot register the constructor with the class object
//you have to define and call the constructor yourself.

typedef void *(*ctor_t)(void);

//a generic iterator. The generic iterator is registered wih the class object
//so that the class object can iterate over the objects.

typedef int (*iter_t)(void *);

//the generic printer is not registered with the class but with the object
//as not all classes/objects need a printer

typedef void (*pter_t)(void *);

typedef struct class
{
  int (*register_object)(struct class *, void *); //the class 'constructor'

  int nobjects_max;
  int nobjects;

  void **objects;

  ctor_t objects_ctor;
  iter_t objects_iter;

  PTHREAD_MUTEX_T objects_mutex; //makes object creation thread-safe
} class_t;

//a class
//objects are derived from a class

//the class 'constructor' register_object registers a created object
//in the class so that the class_iterator can loop over all objects and
//it returns the object_id.
//as the class keeps track of all created objects the class could also delete
//them all when they are no longer needed,
//but this is not needed for the main objects of my program
//object_id can be used to create logical names for derived properties such as
//log-0.txt for the log-file of the first thread object
//log-1.txt for the log-file of the second thread object
//etc.
//you call the class 'constructor' from the object constructor

local int register_object(class_t *self, void *object)
{
  PTHREAD_MUTEX_LOCK(self->objects_mutex);

  BUG(self->nobjects >= self->nobjects_max)

  int iobject = self->nobjects;

  //keep track of objects in class

  self->objects[iobject] = object;

  self->nobjects++;

  PTHREAD_MUTEX_UNLOCK(self->objects_mutex);

  return(iobject);
}

//you initialize a class by calling init_class
//ctor is a pointer to the constructor for objects of the class
//iter is a pointer to the iterator and is called when iterating over all objects of the class
//init_class registers the class 'constructor' register_object

class_t *init_class(int nobjects_max, ctor_t ctor, iter_t iter)
{
  class_t *self;

  MALLOC(self, class_t, 1)

  //the class keeps track of the (number of) created objects

  self->nobjects_max = nobjects_max;

  self->nobjects = 0;

  MALLOC(self->objects, void *, nobjects_max)

  //register the class 'constructor'

  self->register_object = register_object;

  //register the object constructor

  self->objects_ctor = ctor;

  //register the object iterator

  self->objects_iter = iter;

  //make the class constructor thread safe

  PTHREAD_MUTEX_INIT(self->objects_mutex);

  return(self);
}

//the object iterator can return an error value

void iterate_class(class_t *self)
{
  BUG(self->objects_iter == NULL)

  int nerrors = 0;

  for (int iobject = 0; iobject < self->nobjects; iobject++)
    nerrors += self->objects_iter(self->objects[iobject]);

  BUG(nerrors > 0)
}
```

## Here is a basic example

```
//objects are derived from the generic class object

local class_t *my_objects;

//define an object with properties and methods
//object_id is set by the class constructor

typedef struct
{
  //generic properties and methods

  int object_id;

  char object_stamp[LINE_MAX];

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

  self->object_id = my_objects->register_object(my_objects, self);

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

## Here is a complete example that combines GWO with another great library: cJSON

```
//.h
typedef struct state
{
  //generic properties and methods

  int object_id;

  //specific properties

  cJSON *cjson_object;

  //specific methods

  pter_t printf_state;

  void (*set_position)(struct state *, char *);
  void (*add_move)(struct state *, char *);
  void (*set_depth)(struct state *, int);
  void (*set_time)(struct state *, int);
  
  int (*get_depth)(struct state *);
  int (*get_time)(struct state *);
} state_t;

void init_states(void);
void test_states(void);

//.c
//the game state is maintained in a cJSON object
//with the following fields
//CJSON_FEN_ID
//CJSON_MOVES_ID
//CJSON_TIME_ID
//CJSON_DEPTH_ID

local class_t *state_objects;

//the object printer

local void printf_state(void *self)
{
  state_t *state = (state_t *) self;

  PRINTF("object_id=%d\n", state->object_id);

  char string[LINE_MAX];

  BUG(cJSON_PrintPreallocated(state->cjson_object, string, LINE_MAX,
                              TRUE) == 0)

  PRINTF("state=%s\n", string);

  PRINTF("depth=%d\n", state->get_depth(state));
  PRINTF("time=%d\n", state->get_time(state));
}

//object methods

local void set_position(state_t *self, char *position)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(self->cjson_object, CJSON_STARTING_POSITION_ID);

  BUG(!cJSON_IsString(cjson_item))

  cJSON_SetStringValue(cjson_item, position);
}

local void add_move(state_t *self, char *move)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(self->cjson_object, CJSON_MOVES_ID);

  BUG(!cJSON_IsArray(cjson_item))

  cJSON *cjson_move = cJSON_CreateString(move);

  BUG(cjson_move == NULL)

  cJSON_AddItemToArray(cjson_item, cjson_move);
}

local void set_depth(state_t *self, int depth)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(self->cjson_object, CJSON_DEPTH_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  cJSON_SetNumberValue(cjson_item, depth);
}

local void set_time(state_t *self, int time)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(self->cjson_object, CJSON_TIME_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  cJSON_SetNumberValue(cjson_item, time);
}

local int get_depth(state_t *self)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(self->cjson_object, CJSON_DEPTH_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  return(round(cJSON_GetNumberValue(cjson_item)));
}

local int get_time(state_t *self)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(self->cjson_object, CJSON_TIME_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  return(round(cJSON_GetNumberValue(cjson_item)));
}

local void *construct_state(void)
{
  state_t *self;
  
  MALLOC(self, state_t, 1)

  self->object_id = state_objects->register_object(state_objects, self);

  self->cjson_object = cJSON_CreateObject();

  BUG(cJSON_AddStringToObject(self->cjson_object, CJSON_STARTING_POSITION_ID,
                              STARTING_POSITION2FEN) == NULL)

  BUG(cJSON_AddArrayToObject(self->cjson_object, CJSON_MOVES_ID) == NULL)

  BUG(cJSON_AddNumberToObject(self->cjson_object, CJSON_DEPTH_ID, 64) == NULL)

  BUG(cJSON_AddNumberToObject(self->cjson_object, CJSON_TIME_ID, 30) == NULL)

  self->printf_state = printf_state;
  self->set_position = set_position;
  self->add_move = add_move;
  self->set_depth = set_depth;
  self->set_time = set_time;

  self->get_depth = get_depth;
  self->get_time = get_time;

  return(self);
}

//the object iterator

local int iterate_state(void *self)
{
  state_t *state = (state_t *) self;

  PRINTF("iterate object_id=%d\n", state->object_id);

  state->printf_state(self);

  return(0);
}

void init_states(void)
{
  state_objects = init_class(1024, construct_state, iterate_state);
}

void test_states(void)
{
  state_t *a = state_objects->objects_ctor();

  a->set_position(a, "[FEN \"W:W28,31,32,35,36,37,38,39,40,42,43,45,47:B3,7,8,11,12,13,15,19,20,21,23,26,29.\"]");

  a->add_move(a, "31-26");

  a->add_move(a, "17-22");

  a->set_depth(a, 32);

  a->set_time(a, 10);

  state_t *b = state_objects->objects_ctor();

  iterate_class(state_objects);
}

```

You call init_states() once from main(). The output of test_states() is:

```
state={
	"starting_position":	"[FEN \"W:W28,31,32,35,36,37,38,39,40,42,43,45,47:B3,7,8,11,12,13,15,19,20,21,23,26,29.\"]",
	"moves":	["31-26", "17-22"],
	"depth":	32,
	"time":	10
}
depth=32
time=10
iterate object_id=1
object_id=1
state={
	"starting_position":	"[FEN \"W:W31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50:B01,02,03,04,05,06,07,08,09,10,11,12,13,14,15,16,17,18,19,20.\"]",
	"moves":	[],
	"depth":	64,
	"time":	30
}
depth=64
time=30
```
