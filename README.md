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
local char *moves_move2string(moves_t *object, int imove)
{
  snprintf(object->move_string, ..);
  return(object->move_string);
}

void create_moves(moves_t *object)
{
  object->nmoves = 0;
  //similar links to functions for gen_moves, do_move and undo_move
  object->move2string = moves_move2string;
}

//in OO style you could also use malloc() to create the object but remember you have to free it later
/*
moves_t *create_moves(void)
{
  moves_t *object;
  
  object = (moves_t *) MALLOC(sizeof(moves_t));
  object->nmoves = 0;
  object->moves2string = moves_moves2string;
  return(object);
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
//.h
//generic class object

//a generic constructor. The generic constructor is registered wih the class object
//so you can initialize an object using class_object->objects_ctor().

typedef void *(*ctor_t)(void);

//a generic destructor. The generic destructor is registered wih the class object
//so you can initialize an object using the syntax class_object->objects_dtor().

typedef void (*dtor_t)(void *);

//a generic iterator. The generic iterator is registered wih the class object
//so that the class object can iterate over the objects.

typedef int (*iter_t)(void *);

//the generic printer is not registered with the class but with the object
//if needed as not all classes/objects need a printer

typedef void (*pter_t)(void *);

typedef struct class
{
  //objects are created by calling
  //class_objects->objects_ctor().
  //objects_ctor() should call register_object() to register an object
  //within the class
  //objects are destroyed by calling
  //class_objects->objects_dtor()
  //objects_dtor() should call deregister_object() to deregister an object
  //from the class
 
  int (*register_object)(struct class *, void *);
  void (*deregister_object)(struct class *, void *);

  int nobjects_max;
  int nobjects;
  int object_id;

  void **objects;

  ctor_t objects_ctor;
  dtor_t objects_dtor;
  iter_t objects_iter;

  PTHREAD_MUTEX_T objects_mutex;
} class_t;

//.c
//objects are derived from a class

//the class 'constructor' register_object registers a created object
//with the class so that the class_iterator can loop over all objects and
//it returns the object_id.
//object_id can be used to create logical names for derived properties such as
//log-0.txt for the log-file of the first thread object
//log-1.txt for the log-file of the second thread object
//etc.
//object_id's are not re-used and the class will keep track of
//at most nobject_max objects
//you should call the class constructor from the object constructor

local int register_object(class_t *self, void *object)
{
  PTHREAD_MUTEX_LOCK(self->objects_mutex);

  BUG(self->nobjects >= self->nobjects_max)

  self->objects[self->nobjects++] = object;

  int object_id = self->object_id++;

  PTHREAD_MUTEX_UNLOCK(self->objects_mutex);

  return(object_id);
}

//the class 'destructor' deregister_object deregisters an object
//in the class

local void deregister_object(class_t *self, void *object)
{
  PTHREAD_MUTEX_LOCK(self->objects_mutex);

  BUG(self->nobjects < 1)

  int iobject;

  for (iobject = 0; iobject < self->nobjects; iobject++)
    if (self->objects[iobject] == object) break;

  BUG(iobject >= self->nobjects) 

  for (int jobject = iobject; jobject < self->nobjects - 1; jobject++)
    self->objects[jobject] = self->objects[jobject + 1];

  self->objects[self->nobjects - 1] = NULL;

  self->nobjects--;

  PTHREAD_MUTEX_UNLOCK(self->objects_mutex);
}

//you initialize a class by calling init_class
//ctor is a pointer to the constructor for objects of the class
//dtor is a pointer to the destructor for objects of the class
//iter is a is called when iterating over all objects of the class
//init_class registers the class constructor register_object
//and the class destructor deregister_object

class_t *init_class(int nobjects_max, ctor_t ctor, dtor_t dtor, iter_t iter)
{
  class_t *self;
 
  MALLOC(self, class_t, 1)

  //the class keeps track of the (number of) created objects

  self->nobjects_max = nobjects_max;

  self->nobjects = 0;

  self->object_id = 0;

  MALLOC(self->objects, void *, nobjects_max)

  for (int iobject = 0; iobject < nobjects_max; iobject++)
   self->objects[iobject] = NULL;

  //register the class 'constructor' and 'destructor'

  self->register_object = register_object;

  self->deregister_object = deregister_object;

  //register the object constructor

  self->objects_ctor = ctor;

  //register the object destructor

  self->objects_dtor = dtor;

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
  {
    BUG(self->objects[iobject] == NULL)

    nerrors += self->objects_iter(self->objects[iobject]);
  }
 
  BUG(nerrors > 0)
}

```
## Here is a basic example
```
//objects are derived from the generic class object

local class_t *my_object_class;

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
  my_object_t *object = self;

  PRINTF("printing my_object object_id=%d\n", object->object_id);

  PRINTF("object_stamp=%s\n", object->object_stamp);
}

//the object constructor

local void *construct_my_object(void)
{
  my_object_t *object;
  
  MALLOC(object, my_object_t, 1)

  //call the class 'constructor'

  object->object_id = my_object_class->register_object(my_object_class, object);

  //construct other parts of the object

  time_t t = time(NULL);
  (void) strftime(object->object_stamp, LINE_MAX,
    "%H:%M:%S-%d/%m/%Y", localtime(&t));

  //register object methods

  object->printf_object = printf_my_object;

  return(object);
}

//the object destructor

local void destroy_my_object(void *self)
{
  my_object_t *object = self;

  PRINTF("destroying my_object object_id=%d\n", object->object_id);

  //call the class 'destructor'

  my_object_class->deregister_object(my_object_class, object);
}

//the object iterator

local int iterate_my_object(void *self)
{
  my_object_t *object = self;

  PRINTF("iterate object_id=%d\n", object->object_id);

  return(0);
}

void test_my_object_class(void)
{
  //initialize the class 

  my_object_class = init_class(3, construct_my_object, destroy_my_object,
                          iterate_my_object);

  my_object_t *a = my_object_class->objects_ctor();

  a->printf_object(a);

  my_object_t *b = my_object_class->objects_ctor();

  b->printf_object(b);

  my_object_t *c = my_object_class->objects_ctor();

  c->printf_object(c);

  PRINTF("iterate from a to c\n");

  iterate_class(my_object_class);

  my_object_class->objects_dtor(a);

  PRINTF("a has been destroyed, b and c should be left\n");

  iterate_class(my_object_class);

  my_object_t *d = my_object_class->objects_ctor();
  
  PRINTF("d has been added, iterate from b to d\n");

  iterate_class(my_object_class);

  my_object_class->objects_dtor(c);

  PRINTF("c has been destroyed, b and d should be left\n");

  iterate_class(my_object_class);

  my_object_t *e = my_object_class->objects_ctor();
  
  PRINTF("e has been added\n");

  iterate_class(my_object_class);
}

```
## Here is a TEMPLATE you can copy. Just change TEMPLATE to the name of the object and start coding!
```
//objects are derived from the generic class object

local class_t *TEMPLATE_class;

//define an object with properties and methods
//object_id is set by the class constructor

typedef struct
{
  //generic properties and methods

  int object_id;

  char object_stamp[LINE_MAX];

  //specific methods

  pter_t printf_object;
} TEMPLATE_t;

//the object printer

local void printf_TEMPLATE(void *self)
{
  TEMPLATE_t *object = self;

  PRINTF("printing TEMPLATE object_id=%d\n", object->object_id);

  PRINTF("object_stamp=%s\n", object->object_stamp);
}

//the object constructor

local void *construct_TEMPLATE(void)
{
  TEMPLATE_t *object;
  
  MALLOC(object, TEMPLATE_t, 1)

  //call the class 'constructor'

  object->object_id = TEMPLATE_class->register_object(TEMPLATE_class, object);

  //construct other parts of the object

  time_t t = time(NULL);
  (void) strftime(object->object_stamp, LINE_MAX,
    "%H:%M:%S-%d/%m/%Y", localtime(&t));

  //register object methods

  object->printf_object = printf_TEMPLATE;

  return(object);
}

//the object destructor

local void destroy_TEMPLATE(void *self)
{
  TEMPLATE_t *object = self;

  PRINTF("destroying TEMPLATE object_id=%d\n", object->object_id);

  //call the class 'destructor'

  TEMPLATE_class->deregister_object(TEMPLATE_class, object);
}

//the object iterator

local int iterate_TEMPLATE(void *self)
{
  TEMPLATE_t *object = self;

  PRINTF("iterate object_id=%d\n", object->object_id);

  return(0);
}

void test_TEMPLATE_class(void)
{
  //initialize the class 

  TEMPLATE_class = init_class(3, construct_TEMPLATE, destroy_TEMPLATE,
                          iterate_TEMPLATE);

  TEMPLATE_t *a = TEMPLATE_class->objects_ctor();

  a->printf_object(a);

  TEMPLATE_t *b = TEMPLATE_class->objects_ctor();

  b->printf_object(b);

  TEMPLATE_t *c = TEMPLATE_class->objects_ctor();

  c->printf_object(c);

  PRINTF("iterate from a to c\n");

  iterate_class(TEMPLATE_class);

  TEMPLATE_class->objects_dtor(a);

  PRINTF("a has been destroyed, b and c should be left\n");

  iterate_class(TEMPLATE_class);

  TEMPLATE_t *d = TEMPLATE_class->objects_ctor();
  
  PRINTF("d has been added, iterate from b to d\n");

  iterate_class(TEMPLATE_class);

  TEMPLATE_class->objects_dtor(c);

  PRINTF("c has been destroyed, b and d should be left\n");

  iterate_class(TEMPLATE_class);

  TEMPLATE_t *e = TEMPLATE_class->objects_ctor();
  
  PRINTF("e has been added\n");

  iterate_class(TEMPLATE_class);
}
```
## Here is a complete example that combines GWO with another great C library: cJSON

```
//.h
//states.c

typedef struct state
{
  //generic properties and methods

  int object_id;

  //specific properties

  cJSON *cjson_object;

  char string[LINE_MAX];

  //specific methods

  pter_t printf_state;

  void (*set_state)(struct state *, char *);
  void (*set_event)(struct state *, char *);
  void (*set_date)(struct state *, char *);
  void (*set_white)(struct state *, char *);
  void (*set_black)(struct state *, char *);
  void (*set_result)(struct state *, char *);
  void (*set_starting_position)(struct state *, char *);
  void (*push_move)(struct state *, char *, char *);
  void (*pop_move)(struct state *);
  void (*set_depth)(struct state *, int);
  void (*set_time)(struct state *, int);
  void (*save)(struct state *, char *);
  void (*save2pdn)(struct state *, char *);

  char *(*get_state)(struct state *);
  char *(*get_event)(struct state *);
  char *(*get_date)(struct state *);
  char *(*get_white)(struct state *);
  char *(*get_black)(struct state *);
  char *(*get_result)(struct state *);
  char *(*get_starting_position)(struct state *);
  cJSON *(*get_moves)(struct state *);
  int (*get_depth)(struct state *);
  int (*get_time)(struct state *);
  void (*load)(struct state *, char *);
} state_t;

//.c
//the game state is maintained in a cJSON object
//with the following fields

class_t *state_class;

//the object printer

local void printf_state(void *self)
{
  state_t *object = self;

  PRINTF("object_id=%d\n", object->object_id);

  PRINTF("state=%s\n", object->get_state(object));
  PRINTF("event=%s\n", object->get_event(object));
  PRINTF("date=%s\n", object->get_date(object));
  PRINTF("white=%s\n", object->get_white(object));
  PRINTF("black=%s\n", object->get_black(object));
  PRINTF("result=%s\n", object->get_result(object));
  PRINTF("starting_position=%s\n", object->get_starting_position(object));
  PRINTF("depth=%d\n", object->get_depth(object));
  PRINTF("time=%d\n", object->get_time(object));
}

//object methods

local void set_state(state_t *object, char *string)
{
  object->cjson_object = cJSON_Parse(string);

  if (object->cjson_object == NULL)
  { 
    const char *error = cJSON_GetErrorPtr();

    if (error != NULL)
      fprintf(stderr, "json error before: %s\n", error);

    FATAL("gwd.json error", EXIT_FAILURE)
  }
}

local void set_event(state_t *object, char *event)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_EVENT_ID);

  BUG(!cJSON_IsString(cjson_item))

  cJSON_SetStringValue(cjson_item, event);
}

local void set_date(state_t *object, char *date)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_DATE_ID);

  BUG(!cJSON_IsString(cjson_item))

  cJSON_SetStringValue(cjson_item, date);
}

local void set_white(state_t *object, char *white)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_WHITE_ID);

  BUG(!cJSON_IsString(cjson_item))

  cJSON_SetStringValue(cjson_item, white);
}

local void set_black(state_t *object, char *black)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_BLACK_ID);

  BUG(!cJSON_IsString(cjson_item))

  cJSON_SetStringValue(cjson_item, black);
}

local void set_result(state_t *object, char *result)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_RESULT_ID);

  BUG(!cJSON_IsString(cjson_item))

  cJSON_SetStringValue(cjson_item, result);
}

local void set_starting_position(state_t *object, char *position)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_STARTING_POSITION_ID);

  BUG(!cJSON_IsString(cjson_item))

  cJSON_SetStringValue(cjson_item, position);
}

local void push_move(state_t *object, char *move, char *comment)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_MOVES_ID);

  BUG(!cJSON_IsArray(cjson_item))

  cJSON *cjson_move = cJSON_CreateObject();
 
  BUG(cjson_move == NULL)

  BUG(cJSON_AddStringToObject(cjson_move, CJSON_MOVE_STRING_ID, move) == NULL)

  if (comment != NULL)
    BUG(cJSON_AddStringToObject(cjson_move, CJSON_COMMENT_STRING_ID,
                                comment) == NULL)

  cJSON_AddItemToArray(cjson_item, cjson_move);
}

local int pop_move(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_MOVES_ID);

  BUG(!cJSON_IsArray(cjson_item))

  int n = cJSON_GetArraySize(cjson_item);

  if (n > 0) cJSON_DeleteItemFromArray(cjson_item, n - 1);

  return(n);
}

local void set_depth(state_t *object, int depth)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_DEPTH_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  cJSON_SetNumberValue(cjson_item, depth);
}

local void set_time(state_t *object, int time)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_TIME_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  cJSON_SetNumberValue(cjson_item, time);
}

local void save(state_t *object, char *name)
{
  FILE *fsave;

  BUG((fsave = fopen(name, "w")) == NULL)

  fprintf(fsave, "%s\n", object->get_state(object));

  FCLOSE(fsave);
}

local void save2pdn(state_t *object, char *pdn)
{
  FILE *fpdn;

  BUG((fpdn = fopen(pdn, "a")) == NULL)

  fprintf(fpdn, "[Event \"%s\"]\n", object->get_event(object));
  fprintf(fpdn, "[Date \"%s\"]\n", object->get_date(object));
  fprintf(fpdn, "[White \"%s\"]\n", object->get_white(object));
  fprintf(fpdn, "[Black \"%s\"]\n", object->get_black(object));
  fprintf(fpdn, "[Result \"%s\"]\n", object->get_result(object));
  fprintf(fpdn, "[FEN \"%s\"]\n\n", object->get_starting_position(object));

  int iply = 0;

  cJSON *game_move;

  cJSON_ArrayForEach(game_move, object->get_moves(object))
  {
    cJSON *move_string = cJSON_GetObjectItem(game_move, CJSON_MOVE_STRING_ID);

    BUG(!cJSON_IsString(move_string))

    cJSON *comment_string = cJSON_GetObjectItem(game_move,
                                                CJSON_COMMENT_STRING_ID);

    if ((iply % 2) == 0) fprintf(fpdn, " %d.", iply / 2 + 1);

    if (comment_string == NULL)
    {
      fprintf(fpdn, " %s", cJSON_GetStringValue(move_string));
    }
    else
    {
      fprintf(fpdn, " %s {%s}", cJSON_GetStringValue(move_string),
                                cJSON_GetStringValue(comment_string));
    } 

    iply++;

    if ((iply > 0) and ((iply % 10) == 0)) fprintf(fpdn, "\n");
  }
  fprintf(fpdn, " %s\n\n", object->get_result(object));

  FCLOSE(fpdn)
}

local char *get_state(state_t *object)
{
  BUG(cJSON_PrintPreallocated(object->cjson_object, object->string, LINE_MAX,
                              FALSE) == 0)

  return(object->string);
}

local char *get_event(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_EVENT_ID);

  BUG(!cJSON_IsString(cjson_item))

  return(cJSON_GetStringValue(cjson_item));
}

local char *get_date(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_DATE_ID);

  BUG(!cJSON_IsString(cjson_item))

  return(cJSON_GetStringValue(cjson_item));
}

local char *get_white(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_WHITE_ID);

  BUG(!cJSON_IsString(cjson_item))

  return(cJSON_GetStringValue(cjson_item));
}

local char *get_black(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_BLACK_ID);

  BUG(!cJSON_IsString(cjson_item))

  return(cJSON_GetStringValue(cjson_item));
}

local char *get_result(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_RESULT_ID);

  BUG(!cJSON_IsString(cjson_item))

  return(cJSON_GetStringValue(cjson_item));
}

local char *get_starting_position(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_STARTING_POSITION_ID);

  BUG(!cJSON_IsString(cjson_item))

  return(cJSON_GetStringValue(cjson_item));
}

local cJSON *get_moves(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_MOVES_ID);

  BUG(!cJSON_IsArray(cjson_item))

  return(cjson_item);
}

local int get_depth(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_DEPTH_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  return(round(cJSON_GetNumberValue(cjson_item)));
}

local int get_time(state_t *object)
{
  cJSON *cjson_item =
    cJSON_GetObjectItem(object->cjson_object, CJSON_TIME_ID);

  BUG(!cJSON_IsNumber(cjson_item))

  return(round(cJSON_GetNumberValue(cjson_item)));
}

local void load(state_t *object, char *name)
{
  FILE *fload;

  BUG((fload = fopen(name, "r")) == NULL)

  char string[LINE_MAX];

  strcpy(string, "");
  
  while(TRUE)
  {
    char line[LINE_MAX];
 
    if (fgets(line, LINE_MAX, fload) == NULL) break;

    size_t n = strlen(line);

    if (n > 0)
    {
      if (line[n - 1] == '\n') line[n - 1] = '\0';
    }
    strcat(string, " ");
    strcat(string, line);
  }
  FCLOSE(fload)

  object->set_state(object, string);
}

local void *construct_state(void)
{
  state_t *object;
  
  MALLOC(object, state_t, 1)

  object->object_id = state_class->register_object(state_class, object);

  object->cjson_object = cJSON_CreateObject();

  char event[LINE_MAX];

  time_t t = time(NULL);

  (void) strftime(event, LINE_MAX, "%Y.%m.%d %H:%M:%S", localtime(&t));

  BUG(cJSON_AddStringToObject(object->cjson_object, CJSON_EVENT_ID, event) ==
      NULL)

  char date[LINE_MAX];

  (void) strftime(date, LINE_MAX, "%Y.%m.%d", localtime(&t));

  BUG(cJSON_AddStringToObject(object->cjson_object, CJSON_DATE_ID, date) == NULL)

  BUG(cJSON_AddStringToObject(object->cjson_object, CJSON_WHITE_ID, "White") ==
      NULL)

  BUG(cJSON_AddStringToObject(object->cjson_object, CJSON_BLACK_ID, "Black") ==
      NULL)

  BUG(cJSON_AddStringToObject(object->cjson_object, CJSON_RESULT_ID, "*") ==
      NULL)

  BUG(cJSON_AddStringToObject(object->cjson_object, CJSON_STARTING_POSITION_ID,
                              STARTING_POSITION2FEN) == NULL)

  BUG(cJSON_AddArrayToObject(object->cjson_object, CJSON_MOVES_ID) == NULL)

  BUG(cJSON_AddNumberToObject(object->cjson_object, CJSON_DEPTH_ID, 64) == NULL)

  BUG(cJSON_AddNumberToObject(object->cjson_object, CJSON_TIME_ID, 30) == NULL)

  object->printf_state = printf_state;

  object->set_state = set_state;
  object->set_event = set_event;
  object->set_date = set_date;
  object->set_white = set_white;
  object->set_black = set_black;
  object->set_result = set_result;
  object->set_starting_position = set_starting_position;
  object->push_move = push_move;
  object->pop_move = pop_move;
  object->set_depth = set_depth;
  object->set_time = set_time;
  object->save = save;
  object->save2pdn = save2pdn;

  object->get_state = get_state;
  object->get_event = get_event;
  object->get_date = get_date;
  object->get_white = get_white;
  object->get_black = get_black;
  object->get_result = get_result;
  object->get_starting_position = get_starting_position;
  object->get_moves = get_moves;
  object->get_depth = get_depth;
  object->get_time = get_time;
  object->load = load;

  return(object);
}

local void destroy_state(void *self)
{
  state_t *object = self;

  cJSON_Delete(object->cjson_object);

  state_class->deregister_object(state_class, object);
}


//the object iterator

local int iterate_state(void *self)
{
  state_t *object = self;

  PRINTF("iterate object_id=%d\n", object->object_id);

  object->printf_state(object);

  return(0);
}

void init_state_class(void)
{
  state_class = init_class(128, construct_state, destroy_state,
                           iterate_state);
}

void test_state_class(void)
{
  state_t *a = state_class->objects_ctor();

  a->set_starting_position(a, "[FEN \"W:W28,31,32,35,36,37,38,39,40,42,43,45,47:B3,7,8,11,12,13,15,19,20,21,23,26,29.\"]");

  a->push_move(a, "31-26", NULL);

  a->push_move(a, "17-22", "{only move}");

  a->set_depth(a, 32);

  a->set_time(a, 10);

  a->printf_state(a);

  a->pop_move(a);

  state_t *b = state_class->objects_ctor();

  iterate_class(state_class);
}
```

You call init_state_class() once from main(). The output of test_state_class() is:

```
state={"event":"2022.12.28 10:26:02","date":"2022.12.28","white":"White","black":"Black","result":"*","FEN":"[FEN \"W:W28,31,32,35,36,37,38,39,40,42,43,45,47:B3,7,8,11,12,13,15,19,20,21,23,26,29.\"]","moves":[{"move_string":"31-26"},{"move_string":"17-22","comment_string":"{only move}"}],"depth":32,"time":10}
event=2022.12.28 10:26:02
date=2022.12.28
white=White
black=Black
result=*
starting_position=[FEN "W:W28,31,32,35,36,37,38,39,40,42,43,45,47:B3,7,8,11,12,13,15,19,20,21,23,26,29."]
depth=32
time=10
iterate object_id=0
object_id=0
state={"event":"2022.12.28 10:26:02","date":"2022.12.28","white":"White","black":"Black","result":"*","FEN":"[FEN \"W:W28,31,32,35,36,37,38,39,40,42,43,45,47:B3,7,8,11,12,13,15,19,20,21,23,26,29.\"]","moves":[{"move_string":"31-26"}],"depth":32,"time":10}
event=2022.12.28 10:26:02
date=2022.12.28
white=White
black=Black
result=*
starting_position=[FEN "W:W28,31,32,35,36,37,38,39,40,42,43,45,47:B3,7,8,11,12,13,15,19,20,21,23,26,29."]
depth=32
time=10
iterate object_id=1
object_id=1
state={"event":"2022.12.28 10:26:02","date":"2022.12.28","white":"White","black":"Black","result":"*","FEN":"[FEN \"W:W31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50:B01,02,03,04,05,06,07,08,09,10,11,12,13,14,15,16,17,18,19,20.\"]","moves":[],"depth":64,"time":30}
event=2022.12.28 10:26:02
date=2022.12.28
white=White
black=Black
result=*
starting_position=[FEN "W:W31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50:B01,02,03,04,05,06,07,08,09,10,11,12,13,14,15,16,17,18,19,20."]
depth=64
time=30

```
