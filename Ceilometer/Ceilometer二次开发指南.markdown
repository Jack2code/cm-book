# Ceilometer二次开发指南

标签（空格分隔）： Ceilometer

---

[TOC]

## 背景
NaaS-Controller需要保存设备性能数据到Ceilometer的数据库中，而且需要支持多指标查询，故需要对Ceilometer源码进行改造。

## 一、保存性能数据
Ceilometer涉及修改的源码如下：

### 1. ceilometer/api/controllers/v2.py
在`v2.py`中的`MetersController`类中增加`post`方法，对应rest为`/v2/meters`，HTTP方法为`POST`，代码如下：
```python
@wsme.validate([Sample])
@wsme_pecan.wsexpose([Sample], body=[Sample])
def post(self, body):
	"""Post a list of new Samples to Ceilometer.

	:param body: a list of samples within the request body.
	"""
	# Note:
	#  1) the above validate decorator seems to do nothing. LP#1220678
	#  2) the mandatory options seems to also do nothing. LP#1227004
	#  3) the body should already be in a list of Sample's LP#1233219
	now = timeutils.utcnow()
	auth_project = acl.get_limited_to_project(pecan.request.headers)
	def_source = pecan.request.cfg.sample_source
	def_project_id = pecan.request.headers.get('X-Project-Id')
	def_user_id = pecan.request.headers.get('X-User-Id')

	for s in body:
		if s['counter_type'] not in sample.TYPES:
			raise wsme.exc.InvalidInput('counter_type', s['counter_type'],
										'The counter type must be: ' +
										', '.join(sample.TYPES))

		s['user_id'] = (s['user_id'] or def_user_id)
		s['project_id'] = (s['project_id'] or def_project_id)
		s['source'] = '%s:%s' % (s['project_id'], (s['source'] or def_source))
		s['timestamp'] = (s['timestamp'] or now)

		if auth_project and auth_project != s['project_id']:
			# non admin user trying to cross post to another project_id
			auth_msg = 'can not post samples to other projects'
			raise wsme.exc.InvalidInput('project_id', s['project_id'],
										auth_msg)

		s['message_id'] = str(uuid.uuid1())
		data = {
			'source': s['source'],
			'counter_name': s['counter_name'],
			'counter_type': s['counter_type'],
			'counter_unit': s['counter_unit'],
			'counter_volume': s['counter_volume'],
			'user_id': s['user_id'],
			'project_id': s['project_id'],
			'resource_id': s['resource_id'],
			'timestamp': s['timestamp'],
			'resource_metadata': s['resource_metadata'],
			'message_id': s['message_id']
		}
		secret = cfg.CONF.publisher_rpc.metering_secret
		data['message_signature'] = rpc.compute_signature(data, secret)
		pecan.request.storage_conn.record_metering_data(data)

	return body
```

其中，`post`方法最后会引用到外部方法`metering_secret`和`compute_signature`，

```python
secret = cfg.CONF.publisher_rpc.metering_secret
data['message_signature'] = rpc.compute_signature(data, secret)
```

因此，需要引用该方法所属的文件`ceilometer/publisher/rpc.py`：

```python
from ceilometer.publisher import rpc
```

## 二、多指标（复杂）查询性能数据
下面，对Ceilometer涉及的源码按文件进行说明：

### 1. ceilometer/api/controllers/v2.py

在`V2Controller`类中增加`QueryController`类， 对应的rest路径为`/v2/query`：

```python
query = QueryController()
```

`QueryController`类中添加指标查询类`QuerySamplesController`，对应rest路径为`/v2/query/samples`，代码如下：

```python
class QueryController(rest.RestController):
    samples = QuerySamplesController()
```

`QuerySamplesController`类，HTTP方法为`POST`代码如下：

```python
class QuerySamplesController(rest.RestController):
    """Provides complex query possibilities for samples."""

    @wsme_pecan.wsexpose([Sample], body=ComplexQuery)
    def post(self, body):
        """Define query for retrieving Sample data.

        :param body: Query rules for the samples to be returned.
        """
        sample_name_mapping = {"resource": "resource_id",
                               "meter": "counter_name",
                               "type": "counter_type",
                               "unit": "counter_unit",
                               "volume": "counter_volume"}

        query = ValidatedComplexQuery(body,
                                      storage.models.Sample,
                                      sample_name_mapping,
                                      metadata_allowed=True)
        query.validate(visibility_field="project_id")
        conn = pecan.request.storage_conn
        return [Sample.from_db_model(s)
                for s in conn.query_samples(query.filter_expr,
                                            query.orderby,
                                            query.limit)]
```

其中，`QuerySamplesController`类中引用`ComplexQuery`类的代码如下：

```python
class ComplexQuery(_Base):
    """Holds a sample query encoded in json."""

    filter = wtypes.text
    "The filter expression encoded in json."

    orderby = wtypes.text
    "List of single-element dicts for specifing the ordering of the results."

    limit = int
    "The maximum number of results to be returned."

    @classmethod
    def sample(cls):
        return cls(filter='{"and": [{"and": [{"=": ' +
                          '{"counter_name": "cpu_util"}}, ' +
                          '{">": {"counter_volume": 0.23}}, ' +
                          '{"<": {"counter_volume": 0.26}}]}, ' +
                          '{"or": [{"and": [{">": ' +
                          '{"timestamp": "2013-12-01T18:00:00"}}, ' +
                          '{"<": ' +
                          '{"timestamp": "2013-12-01T18:15:00"}}]}, ' +
                          '{"and": [{">": ' +
                          '{"timestamp": "2013-12-01T18:30:00"}}, ' +
                          '{"<": ' +
                          '{"timestamp": "2013-12-01T18:45:00"}}]}]}]}',
                   orderby='[{"counter_volume": "ASC"}, ' +
                           '{"timestamp": "DESC"}]',
                   limit=42
                   )
```

`QuerySamplesController`类中引用`ValidatedComplexQuery`类的代码如下：
```python
class ValidatedComplexQuery(object):
    complex_operators = ["and", "or"]
    order_directions = ["asc", "desc"]
    simple_ops = ["=", "!=", "<", ">", "<=", "=<", ">=", "=>"]
    regexp_prefix = "(?i)"

    complex_ops = _list_to_regexp(complex_operators, regexp_prefix)
    simple_ops = _list_to_regexp(simple_ops, regexp_prefix)
    order_directions = _list_to_regexp(order_directions, regexp_prefix)

    timestamp_fields = ["timestamp", "state_timestamp"]

    def __init__(self, query, db_model, additional_name_mapping=None,
                 metadata_allowed=False):
        additional_name_mapping = additional_name_mapping or {}
        self.name_mapping = {"user": "user_id",
                             "project": "project_id"}
        self.name_mapping.update(additional_name_mapping)
        valid_keys = db_model.get_field_names()
        valid_keys = list(valid_keys) + self.name_mapping.keys()
        valid_fields = _list_to_regexp(valid_keys)

        if metadata_allowed:
            valid_filter_fields = valid_fields + "|^metadata\.[\S]+$"
        else:
            valid_filter_fields = valid_fields

        schema_value = {
            "oneOf": [{"type": "string"},
                      {"type": "number"},
                      {"type": "boolean"}],
            "minProperties": 1,
            "maxProperties": 1}

        schema_value_in = {
            "type": "array",
            "items": {"oneOf": [{"type": "string"},
                                {"type": "number"}]},
            "minItems": 1}

        schema_field = {
            "type": "object",
            "patternProperties": {valid_filter_fields: schema_value},
            "additionalProperties": False,
            "minProperties": 1,
            "maxProperties": 1}

        schema_field_in = {
            "type": "object",
            "patternProperties": {valid_filter_fields: schema_value_in},
            "additionalProperties": False,
            "minProperties": 1,
            "maxProperties": 1}

        schema_leaf_in = {
            "type": "object",
            "patternProperties": {"(?i)^in$": schema_field_in},
            "additionalProperties": False,
            "minProperties": 1,
            "maxProperties": 1}

        schema_leaf_simple_ops = {
            "type": "object",
            "patternProperties": {self.simple_ops: schema_field},
            "additionalProperties": False,
            "minProperties": 1,
            "maxProperties": 1}

        schema_and_or_array = {
            "type": "array",
            "items": {"$ref": "#"},
            "minItems": 2}

        schema_and_or = {
            "type": "object",
            "patternProperties": {self.complex_ops: schema_and_or_array},
            "additionalProperties": False,
            "minProperties": 1,
            "maxProperties": 1}

        schema_not = {
            "type": "object",
            "patternProperties": {"(?i)^not$": {"$ref": "#"}},
            "additionalProperties": False,
            "minProperties": 1,
            "maxProperties": 1}

        self.schema = {
            "oneOf": [{"$ref": "#/definitions/leaf_simple_ops"},
                      {"$ref": "#/definitions/leaf_in"},
                      {"$ref": "#/definitions/and_or"},
                      {"$ref": "#/definitions/not"}],
            "minProperties": 1,
            "maxProperties": 1,
            "definitions": {"leaf_simple_ops": schema_leaf_simple_ops,
                            "leaf_in": schema_leaf_in,
                            "and_or": schema_and_or,
                            "not": schema_not}}

        self.orderby_schema = {
            "type": "array",
            "items": {
                "type": "object",
                "patternProperties":
                    {valid_fields:
                        {"type": "string",
                         "pattern": self.order_directions}},
                "additionalProperties": False,
                "minProperties": 1,
                "maxProperties": 1}}

        self.original_query = query

    def validate(self, visibility_field):
        """Validates the query content and does the necessary conversions."""
        if self.original_query.filter is wtypes.Unset:
            self.filter_expr = None
        else:
            try:
                self.filter_expr = json.loads(self.original_query.filter)
                self._validate_filter(self.filter_expr)
            except (ValueError, jsonschema.exceptions.ValidationError) as e:
                raise ClientSideError(_("Filter expression not valid: %s") %
                                      e.message)
            self._replace_isotime_with_datetime(self.filter_expr)
            self._convert_operator_to_lower_case(self.filter_expr)
            self._normalize_field_names_for_db_model(self.filter_expr)

        self._force_visibility(visibility_field)

        if self.original_query.orderby is wtypes.Unset:
            self.orderby = None
        else:
            try:
                self.orderby = json.loads(self.original_query.orderby)
                self._validate_orderby(self.orderby)
            except (ValueError, jsonschema.exceptions.ValidationError) as e:
                raise ClientSideError(_("Order-by expression not valid: %s") %
                                      e.message)
            self._convert_orderby_to_lower_case(self.orderby)
            self._normalize_field_names_in_orderby(self.orderby)

        if self.original_query.limit is wtypes.Unset:
            self.limit = None
        else:
            self.limit = self.original_query.limit

        if self.limit is not None and self.limit <= 0:
            msg = _('Limit should be positive')
            raise ClientSideError(msg)

    @staticmethod
    def _convert_orderby_to_lower_case(orderby):
        for orderby_field in orderby:
            utils.lowercase_values(orderby_field)

    def _normalize_field_names_in_orderby(self, orderby):
        for orderby_field in orderby:
            self._replace_field_names(orderby_field)

    def _traverse_postorder(self, tree, visitor):
        op = tree.keys()[0]
        if op.lower() in self.complex_operators:
            for i, operand in enumerate(tree[op]):
                self._traverse_postorder(operand, visitor)
        if op.lower() == "not":
            self._traverse_postorder(tree[op], visitor)

        visitor(tree)

    def _check_cross_project_references(self, own_project_id,
                                        visibility_field):
        """Do not allow other than own_project_id."""
        def check_project_id(subfilter):
            op = subfilter.keys()[0]
            if (op.lower() not in self.complex_operators
                    and subfilter[op].keys()[0] == visibility_field
                    and subfilter[op][visibility_field] != own_project_id):
                raise ProjectNotAuthorized(subfilter[op][visibility_field])

        self._traverse_postorder(self.filter_expr, check_project_id)

    def _force_visibility(self, visibility_field):
        """Force visibility field.

        If the tenant is not admin insert an extra
        "and <visibility_field>=<tenant's project_id>" clause to the query.
        """
        authorized_project = acl.get_limited_to_project(pecan.request.headers)
        is_admin = authorized_project is None
        if not is_admin:
            self._restrict_to_project(authorized_project, visibility_field)
            self._check_cross_project_references(authorized_project,
                                                 visibility_field)

    def _restrict_to_project(self, project_id, visibility_field):
        restriction = {"=": {visibility_field: project_id}}
        if self.filter_expr is None:
            self.filter_expr = restriction
        else:
            self.filter_expr = {"and": [restriction, self.filter_expr]}

    def _replace_isotime_with_datetime(self, filter_expr):
        def replace_isotime(subfilter):
            op = subfilter.keys()[0]
            if (op.lower() not in self.complex_operators
                    and subfilter[op].keys()[0] in self.timestamp_fields):
                field = subfilter[op].keys()[0]
                date_time = self._convert_to_datetime(subfilter[op][field])
                subfilter[op][field] = date_time

        self._traverse_postorder(filter_expr, replace_isotime)

    def _normalize_field_names_for_db_model(self, filter_expr):
        def _normalize_field_names(subfilter):
            op = subfilter.keys()[0]
            if op.lower() not in self.complex_operators:
                self._replace_field_names(subfilter.values()[0])
        self._traverse_postorder(filter_expr,
                                 _normalize_field_names)

    def _replace_field_names(self, subfilter):
        field = subfilter.keys()[0]
        value = subfilter[field]
        if field in self.name_mapping:
            del subfilter[field]
            subfilter[self.name_mapping[field]] = value
        if field.startswith("metadata."):
            del subfilter[field]
            subfilter["resource_" + field] = value

    def _convert_operator_to_lower_case(self, filter_expr):
        self._traverse_postorder(filter_expr, utils.lowercase_keys)

    @staticmethod
    def _convert_to_datetime(isotime):
        try:
            date_time = timeutils.parse_isotime(isotime)
            date_time = date_time.replace(tzinfo=None)
            return date_time
        except ValueError:
            LOG.exception(_("String %s is not a valid isotime") % isotime)
            msg = _('Failed to parse the timestamp value %s') % isotime
            raise ClientSideError(msg)

    def _validate_filter(self, filter_expr):
        jsonschema.validate(filter_expr, self.schema)

    def _validate_orderby(self, orderby_expr):
        jsonschema.validate(orderby_expr, self.orderby_schema)
```

在`ValidatedComplexQuery`类中使用的方法`_list_to_regexp`的代码如下：

```python
def _list_to_regexp(items, regexp_prefix=""):
    regexp = ["^%s$" % item for item in items]
    regexp = regexp_prefix + "|".join(regexp)
    return regexp
```

在`ValidatedComplexQuery`中使用`jsonschema`包的路径`/usr/lib64/python2.6/site-packages/jsonschema`，代码如下：

```python
import jsonschema
```

在`ValidatedComplexQuery`中的方法会使用到`ClientSideError`，需要在`v2.py`中添加`ClientSideError`类代码如下：

```python
class ClientSideError(wsme.exc.ClientSideError):
    def __init__(self, error):
        pecan.response.translatable_error = error
        super(ClientSideError, self).__init__(error)
```

### 2. ceilometer/storagy/models.py
在`Model`类中添加类方法`get_field_names`，如下：

```python
@classmethod
def get_field_names(cls):
    fields = inspect.getargspec(cls.__init__)[0]
    return set(fields) - set(["self"])
```

需要导入`inspect`，其路径为：`/usr/lib64/python2.6/inspect.py`：

```python
import inspect
```

### 3. ceilometer/utils.py
添加工具方法，如下：

```python
def lowercase_keys(mapping):
    """Converts the values of the keys in mapping to lowercase."""
    items = mapping.items()
    for key, value in items:
        del mapping[key]
        mapping[key.lower()] = value

def lowercase_values(mapping):
    """Converts the values in the mapping dict to lowercase."""
    items = mapping.items()
    for key, value in items:
        mapping[key] = value.lower() 
```

### 4. ceilometer/storage/base.py
在`Connection`类中添加方法`query_samples`，代码如下：

```python
@abc.abstractmethod
def query_samples(self, filter_expr=None, orderby=None, limit=None):
    """Return an iterable of model.Sample objects.

    :param filter_expr: Filter expression for query.
    :param orderby: List of field name and direction pairs for order by.
    :param limit: Maximum number of results to return.
    """
```
                                             
### 5. ceilometer/storage/impl_mongodb.py
添加`QueryTransformer`类：

```python
class QueryTransformer(object):

    operators = {"<": "$lt",
                 ">": "$gt",
                 "<=": "$lte",
                 "=<": "$lte",
                 ">=": "$gte",
                 "=>": "$gte",
                 "!=": "$ne",
                 "in": "$in"}

    complex_operators = {"or": "$or",
                         "and": "$and"}

    ordering_functions = {"asc": pymongo.ASCENDING,
                          "desc": pymongo.DESCENDING}

    def transform_orderby(self, orderby):
        orderby_filter = []

        for field in orderby:
            field_name = field.keys()[0]
            ordering = self.ordering_functions[field.values()[0]]
            orderby_filter.append((field_name, ordering))
        return orderby_filter

    @staticmethod
    def _move_negation_to_leaf(condition):
        """Moves every not operator to the leafs.

        Moving is going by applying the De Morgan rules and annihilating
        double negations.
        """
        def _apply_de_morgan(tree, negated_subtree, negated_op):
            if negated_op == "and":
                new_op = "or"
            else:
                new_op = "and"

            tree[new_op] = [{"not": child}
                            for child in negated_subtree[negated_op]]
            del tree["not"]

        def transform(subtree):
            op = subtree.keys()[0]
            if op in ["and", "or"]:
                [transform(child) for child in subtree[op]]
            elif op == "not":
                negated_tree = subtree[op]
                negated_op = negated_tree.keys()[0]
                if negated_op == "and":
                    _apply_de_morgan(subtree, negated_tree, negated_op)
                    transform(subtree)
                elif negated_op == "or":
                    _apply_de_morgan(subtree, negated_tree, negated_op)
                    transform(subtree)
                elif negated_op == "not":
                    # two consecutive not annihilates themselves
                    new_op = negated_tree.values()[0].keys()[0]
                    subtree[new_op] = negated_tree[negated_op][new_op]
                    del subtree["not"]
                    transform(subtree)

        transform(condition)

    def transform_filter(self, condition):
        # in Mongo not operator can only be applied to
        # simple expressions so we have to move every
        # not operator to the leafs of the expression tree
        self._move_negation_to_leaf(condition)
        return self._process_json_tree(condition)

    def _handle_complex_op(self, complex_op, nodes):
        element_list = []
        for node in nodes:
            element = self._process_json_tree(node)
            element_list.append(element)
        complex_operator = self.complex_operators[complex_op]
        op = {complex_operator: element_list}
        return op

    def _handle_not_op(self, negated_tree):
        # assumes that not is moved to the leaf already
        # so we are next to a leaf
        negated_op = negated_tree.keys()[0]
        negated_field = negated_tree[negated_op].keys()[0]
        value = negated_tree[negated_op][negated_field]
        if negated_op == "=":
            return {negated_field: {"$ne": value}}
        elif negated_op == "!=":
            return {negated_field: value}
        else:
            return {negated_field: {"$not":
                                    {self.operators[negated_op]: value}}}

    def _handle_simple_op(self, simple_op, nodes):
        field_name = nodes.keys()[0]
        field_value = nodes.values()[0]

        # no operator for equal in Mongo
        if simple_op == "=":
            op = {field_name: field_value}
            return op

        operator = self.operators[simple_op]
        op = {field_name: {operator: field_value}}
        return op

    def _process_json_tree(self, condition_tree):
        operator_node = condition_tree.keys()[0]
        nodes = condition_tree.values()[0]

        if operator_node in self.complex_operators:
            return self._handle_complex_op(operator_node, nodes)

        if operator_node == "not":
            negated_tree = condition_tree[operator_node]
            return self._handle_not_op(negated_tree)

        return self._handle_simple_op(operator_node, nodes)
```

在`Connection`类中添加如下方法：

```python
def query_samples(self, filter_expr=None, orderby=None, limit=None):
    if limit == 0:
        return []
    query_filter = {}
    orderby_filter = [("timestamp", pymongo.DESCENDING)]
    transformer = QueryTransformer()
    if orderby is not None:
        orderby_filter = transformer.transform_orderby(orderby)
    if filter_expr is not None:
        query_filter = transformer.transform_filter(filter_expr)

    return self._retrieve_samples(query_filter, orderby_filter, limit)        

def _retrieve_samples(self, query, orderby, limit):
    if limit is not None:
        samples = self.db.meter.find(query,
                                     limit=limit,
                                     sort=orderby)
    else:
        samples = self.db.meter.find(query,
                                     sort=orderby)

    for s in samples:
        # Remove the ObjectId generated by the database when
        # the sample was inserted. It is an implementation
        # detail that should not leak outside of the driver.
        del s['_id']
        # Backward compatibility for samples without units
        s['counter_unit'] = s.get('counter_unit', '')
        # Tolerate absence of recorded_at in older datapoints
        #s['recorded_at'] = s.get('recorded_at')
        # Check samples for metadata and "unquote" key if initially it
        # was started with '$'.
        if s.get('resource_metadata'):
            s['resource_metadata'] = unquote_keys(
                s.get('resource_metadata'))
        yield models.Sample(**s)
```
            
`_retrieve_samples`方法中引用了`unquote_keys`方法，在`impl_mongodb.py`中添加方法`unquote_keys`：

```python
def unquote_keys(data):
    """Restores initial view of 'quoted' keys in dictionary data

    :param data: is a dictionary
    :return: data with restored keys if they were 'quoted'.
    """
    if isinstance(data, dict):
        for key, value in data.items():
            if isinstance(value, dict):
                unquote_keys(value)
            if key.startswith('%24'):
                k = parse.unquote(key)
                data[k] = data.pop(key)
    return data
```    
## 三、重启服务

```python
service openstack-ceilometer-agent-central restart
service openstack-ceilometer-agent-compute restart
service openstack-ceilometer-collector restart
service openstack-ceilometer-api restart
```