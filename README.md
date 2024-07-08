import { Button, Select, Spin, Tag, message } from 'antd';
import { useContext, useEffect, useState, useCallback } from 'react';
import LangContext from '../../../helps/contexts/lang-context';
import GetApi from '../../../helps/contexts/Api/GetApi';
import PostApi from '../../../helps/contexts/Api/PostApi';
import DeleteApi from '../../../helps/contexts/Api/DeleteApi';
const { Option } = Select;

const EditTagName = ({ onCancel, playerTags, id }) => {
  const ctx = useContext(LangContext);
  const [loading, setLoading] = useState(false);
  const [tagNameList, setTagNameList] = useState([]);
  const [searchTerm, setSearchTerm] = useState('');
  const lang = ctx.lang;
  const [selectedTagNames, setSelectedTagNames] = useState(
    playerTags.map((tag) => ({ value: tag?.id, label: tag?.name }))
  );
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  const fetchTagNameList = useCallback(async (page, searchTerm) => {
    setLoading(true);
    try {
      const path = `/admin/tags`;
      const params = { page };
      if (searchTerm) params['tag_name'] = searchTerm;

      const response = await GetApi.sendApiRequest(path, {}, params);
      if (response.status) {
        setTagNameList((prevTags) => {
          if (searchTerm) {
            return response?.tags?.data || [];
          } else {
            const existingTagIds = new Set(prevTags.map((tag) => tag.id));
            const newTags = response?.tags?.data.filter((tag) => !existingTagIds.has(tag.id));
            return [...prevTags, ...newTags];
          }
        });
        setHasMore(response.tags.current_page < response.tags.last_page);
      }
    } catch (error) {
      console.error('Failed to fetch tags', error);
    } finally {
      setLoading(false);
    }
  }, []);

  const debounceFetchTags = useCallback(
    (searchTerm) => {
      const timeoutId = setTimeout(() => fetchTagNameList(page, searchTerm), 300);
      return () => clearTimeout(timeoutId);
    },
    [page, fetchTagNameList]
  );

  useEffect(() => {
    if (!searchTerm) {
      fetchTagNameList(page, '');
    } else {
      debounceFetchTags(searchTerm);
    }
  }, [page, searchTerm]);

  const handlePopupScroll = (event) => {
    const { target } = event;
    if (target.scrollTop + target.clientHeight >= target.scrollHeight && !loading && hasMore) {
      setPage((prevPage) => prevPage + 1);
    }
  };

  const handleChange = (value) => {
    debounceFetchTags('');
    setSelectedTagNames(value);
  };

  const handleCancel = () => {
    const list = playerTags.map((tag) => ({ value: tag?.id, label: tag?.name }));
    setSelectedTagNames(list);
  };

  useEffect(() => {
    const list = playerTags.map((tag) => ({ value: tag?.id, label: tag?.name }));
    setSelectedTagNames(list);
  }, [playerTags]);

  const onSubmit = async (e) => {
    e.preventDefault();
    const currentPlayerIds = playerTags?.map((item) => item?.id);
    const selectedIds = selectedTagNames?.map((item) => item?.value);
    const removedTagIds = currentPlayerIds.filter((item) => !selectedIds.includes(item));
    const newTagNameIds = selectedTagNames.map((item) => item?.value);
    try {
      if (removedTagIds?.length) {
        const delRes = await DeleteApi.DeleteApiRequest1('/admin/tags/player/remove', {
          tags: removedTagIds, // Array of tag IDs to remove
          player_id: id // Player ID
        });
        if (delRes && delRes.status === 200) {
          message.success('Tag Name removed successfully');
        } else {
          message.error('Failed to remove Tag name');
        }
      } else {
        const res = await PostApi.PostApiRequest('/admin/tags/player/assign', {
          tags: newTagNameIds,
          player_id: id
        });
        if (res && res.status === 200) {
          message.success('Tag Name added successfully');
        } else {
          message.error('Failed to add Tag name');
        }
      }
    } catch (e) {
      console.error('Some error', e.status);
      if (e.status === 409) {
        message.error('Tag names already exist');
      } else {
        message.error('An error occurred while adding tag name');
      }
    } finally {
      setPage(1);
      handleCancel();
      onCancel();
    }
  };

  return (
    <div style={{ height: '155px' }}>
      <form onSubmit={onSubmit}>
        <p style={{ color: '#4F5057' }}>{lang.label_tag_name}</p>

        <Select
          mode="multiple"
          style={{ width: '450px', color: '#4F5057' }}
          placeholder="Select Tag Name"
          // notFoundContent={loading ? <Spin size="small" /> : null}
          onChange={handleChange}
          labelInValue
          value={selectedTagNames}
          onSearch={debounceFetchTags}
          filterOption={false}
          onPopupScroll={handlePopupScroll}
          dropdownRender={(menu) => (
            <div>
              {loading ? (
                <div style={{ padding: '8px', textAlign: 'center' }}>
                  <Spin size="small" />
                </div>
              ) : null}
              {menu}
            </div>
          )}>
          {tagNameList.map((tagname) => (
            <Option key={tagname.id} value={tagname.id} label={tagname.name}>
              {tagname.name}
            </Option>
          ))}
        </Select>

        <span
          style={{
            float: 'right'
          }}>
          <Button
            style={{
              borderRadius: '3px',
              backgroundColor: '#004A7F',
              color: 'white',
              position: 'absolute',
              right: '30%',
              bottom: '15px',
              margin: '-1%'
            }}
            htmlType="submit">
            {lang.label_submit}
          </Button>
          <Button
            style={{
              borderRadius: '3px',
              color: '#004A7F',
              position: 'absolute',
              right: '8%',
              bottom: '15px',
              margin: '-1%'
            }}
            onClick={() => {
              handleCancel();
              onCancel();
              setPage(1);
            }}>
            {lang.label_cancel}
          </Button>
        </span>
      </form>
    </div>
  );
};

export default EditTagName;
