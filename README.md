import React, { useState, useCallback, useEffect } from "react";
import {
  TextField,
  Button,
  Box,
  Container,
  Paper,
  Typography,
  IconButton,
  Grid,
  Card,
  CardContent,
  Divider,
  Dialog,
  DialogTitle,
  DialogContent,
  DialogActions,
  InputAdornment,
} from "@mui/material";
import { DataGrid } from "@mui/x-data-grid";
import HistoryIcon from "@mui/icons-material/History";
import CloseIcon from "@mui/icons-material/Close";
import SearchIcon from "@mui/icons-material/Search";
import useApi from "../../hooks/useApi";
const BalanceParticularSearchScreen = () => {
  const { callApi } = useApi();
  // Filters state including all search inputs
  const [filters, setFilters] = useState({
    date: "",
    branchCode: "",
    currency: "",
    cgl: "",
  });

  const [rows, setRows] = useState([]);
  const [loading, setLoading] = useState(false);
  const [totalElements, setTotalElements] = useState(0);
const[etlDate,setEtlDate]=useState();
  // Pagination state: pageSize default 10, page default 0
  const [paginationModel, setPaginationModel] = useState({
    page: 0,
    pageSize: 10,
  });

  const [openHistory, setOpenHistory] = useState(false);
  const [selectedRow, setSelectedRow] = useState(null);

  const updateFilter = (key, value) =>
    setFilters((prev) => ({ ...prev, [key]: value }));

  // 1. Fetch Default ETL Date on Page Load
  useEffect(() => {
    const fetchDefaultETLDate = async () => {
      try {
        const response = await callApi("/PS/file/fincore-date", {}, "GET");
        const etlDate = response?.data?.etlDate || response?.date || "";
        if (etlDate) {
          setEtlDate(etlDate);
          updateFilter("date", etlDate);
          // Initial fetch with default date and pagination
          fetchData(etlDate, 0, paginationModel.pageSize);
        }
      } catch (err) {
        console.error("ETL Date Error:", err);
      }
    };
    fetchDefaultETLDate();
  }, []);

  // 2. Main Search API Call (Server-side Pagination)
  const fetchData = useCallback(
    async (date, pageIndex, pageSize) => {
      if (!date) return;
      setLoading(true);
      try {
        // Sending separate fields in payload as requested
        const payload = {
          fromDate: date,
          toDate: date,
          branchCode: filters.branchCode || null,
          currency: filters.currency || null,
          cgl: filters.cgl || null,
        };

        const response = await callApi(
          `/ES/differences/search?page=${pageIndex}&size=${pageSize}`,
          payload,
          "POST"
        );
        console.log("HINHIHIHIHIH", payload)
        const content = response?.data?.content || response?.content || [];
        const mappedRows = content.map((item, index) => ({
          id: item.id || `row-${pageIndex}-${index}`,
          ...item,
          GLCC_VAL: `${item.branchCode} ${item.currency} ${item.cgl}`,
        }));

        setRows(mappedRows);
        setTotalElements(
          response?.data?.totalElements || response?.totalElements || 0
        );
      } catch (err) {
        console.error("Search Fail:", err);
        setRows([]);
      } finally {
        setLoading(false);
      }
    },
    [filters]
  );

 
  useEffect(() => {
    if (filters.date) {
      fetchData(filters.date, paginationModel.page, paginationModel.pageSize);
    }
  }, [paginationModel.page, paginationModel.pageSize]);

  const columns = [
    { field: "reconRunDate", headerName: "Error Date" },
    { field: "GLCC_VAL", headerName: "GLCC" },
    { field: "cbsBalance", headerName: "CBS Balance", align: "right" },
    { field: "glBalance", headerName: "GL Balance", align: "right" },
    {
      field: "differenceAmount",
      headerName: "Difference",
      align: "right",
      renderCell: (params) => (
        <Typography
          sx={{
            color: params.value < 0 ? "#d32f2f" : "#2e7d32",
            fontWeight: 700,
          }}
        >
          {params.value?.toLocaleString()}
        </Typography>
      ),
    },
    {
      field: "action",
      headerName: "Action",
      sortable: false,
      renderCell: (params) => (
        <IconButton
          color="primary"
          onClick={() => {
            setSelectedRow(params.row);
            setOpenHistory(true);
          }}
        >
          <HistoryIcon />
        </IconButton>
      ),
    },
  ];

  return (
    <Container maxWidth="xl" sx={{ mt: 4 }}>
     

      {/* Search Filter Section */}
      <Card
        sx={{
          mb: 4,
          boxShadow: "0 4px 12px rgba(0,0,0,0.05)",
        }}
      >
        <CardContent sx={{ p: 3 }}>
          <Grid container spacing={10}>
            <Grid item xs={12} sm={6} md={2.5}>
              <TextField
                label="ETL Date"
                type="date"
                size="small"
                value={etlDate}
                onChange={(e) => { setEtlDate(e.target.value);
                   updateFilter("date", e.target.value);}
                }
                InputLabelProps={{ shrink: true }}
              />
            </Grid>
            <Grid item xs={12} sm={3} md={1.5}>
              <TextField
                fullWidth
                label="Branch"
                size="small"
                value={filters.branchCode}
                onChange={(e) => updateFilter("branchCode", e.target.value)}
              />
            </Grid>
            <Grid item xs={12} sm={3} md={1.5}>
              <TextField
                fullWidth
                label="Currency"
                size="small"
                value={filters.currency}
                onChange={(e) => updateFilter("currency", e.target.value)}
              />
            </Grid>
            <Grid item xs={12} sm={8} md={4.5}>
              <TextField
                fullWidth
                label="CGL Number"
                size="small"
                placeholder="Search by CGL..."
                value={filters.cgl}
                onChange={(e) => updateFilter("cgl", e.target.value)}
                InputProps={{
                  startAdornment: (
                    <InputAdornment position="start">
                      <SearchIcon fontSize="small" />
                    </InputAdornment>
                  ),
                }}
              />
            </Grid>
            <Grid item xs={12} sm={4} md={2}>
              <Button
                fullWidth
                variant="contained"
                sx={{
                  // height: 40,
                  bgcolor: "#1a237e",
                  padding: "10px 40px",
                }}
                onClick={() => {
                  setPaginationModel((prev) => ({ ...prev, page: 0 })); // Reset to page 0 on new search
                  fetchData(filters.date, 0, paginationModel.pageSize);
                }}
              >
                Search
              </Button>
            </Grid>
          </Grid>
        </CardContent>
      </Card>

      {/* Table Section with Pagination Dropdown */}
      <Paper sx={{ height: 550, width: "100%", overflow: "hidden" }}>
        <DataGrid
          rows={rows}
          columns={columns}
          rowCount={totalElements}
          loading={loading}
          paginationMode="server"
          paginationModel={paginationModel}
          onPaginationModelChange={setPaginationModel}
          pageSizeOptions={[10, 25, 50, 100]}
          disableSelectionOnClick
        />
      </Paper>

      {/* Dialog UI for History/Details */}
      <Dialog
        open={openHistory}
        onClose={() => setOpenHistory(false)}
        maxWidth="md"
        fullWidth
        PaperProps={{ sx: { borderRadius: 4, p: 1 } }}
      >
        <DialogTitle
          sx={{
            m: 0,
            p: 2,
            display: "flex",
            justifyContent: "space-between",
            alignItems: "center",
          }}
        >
          <Typography variant="h6" sx={{ fontWeight: 700, color: "#374151" }}>
            Difference Details
          </Typography>
          <IconButton onClick={() => setOpenHistory(false)}>
            <CloseIcon />
          </IconButton>
        </DialogTitle>

        <DialogContent dividers sx={{ border: "none" }}>
          {selectedRow && (
            <Box sx={{ py: 2 }}>
              <Grid container spacing={4} sx={{ mb: 4 }}>
                <Grid item xs={3}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    BRANCH CODE
                  </Typography>
                  <Typography variant="body1" sx={{ fontWeight: 500, mt: 0.5 }}>
                    {selectedRow.branchCode}
                  </Typography>
                </Grid>
                <Grid item xs={3}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    CURRENCY
                  </Typography>
                  <Typography variant="body1" sx={{ fontWeight: 500, mt: 0.5 }}>
                    {selectedRow.currency}
                  </Typography>
                </Grid>
                <Grid item xs={3}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    GLCC COMBINATION
                  </Typography>
                  <Typography variant="body1" sx={{ fontWeight: 500, mt: 0.5 }}>
                    {selectedRow.GLCC_VAL}
                  </Typography>
                </Grid>
                <Grid item xs={3}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    RECON DATE
                  </Typography>
                  <Typography variant="body1" sx={{ fontWeight: 500, mt: 0.5 }}>
                    {selectedRow.reconRunDate}
                  </Typography>
                </Grid>
              </Grid>

              <Grid container spacing={4}>
                <Grid item xs={3}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    CBS BALANCE
                  </Typography>
                  <Typography variant="body1" sx={{ mt: 0.5 }}>
                    {selectedRow.cbsBalance?.toLocaleString()}
                  </Typography>
                </Grid>
                <Grid item xs={3}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    GL BALANCE
                  </Typography>
                  <Typography variant="body1" sx={{ mt: 0.5 }}>
                    {selectedRow.glBalance?.toLocaleString()}
                  </Typography>
                </Grid>
                <Grid item xs={6}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    HEAD / TYPE
                  </Typography>
                  <Typography
                    variant="body1"
                    sx={{ mt: 0.5, fontStyle: "italic", color: "#1a237e" }}
                  >
                    {selectedRow.head} | {selectedRow.type}
                  </Typography>
                </Grid>
              </Grid>

              <Divider sx={{ my: 3 }} />

              <Box
                display="flex"
                justifyContent="space-between"
                alignItems="center"
              >
                <Box>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    YESTERDAY DIFFERENCE
                  </Typography>
                  <Typography variant="h6">
                    {selectedRow.diffBwYesterday?.toLocaleString()}
                  </Typography>
                </Box>
                <Box sx={{ textAlign: "right" }}>
                  <Typography
                    variant="caption"
                    color="text.secondary"
                    sx={{ fontWeight: 600 }}
                  >
                    TOTAL DIFFERENCE AMOUNT
                  </Typography>
                  <Typography
                    variant="h5"
                    sx={{
                      fontWeight: 800,
                      color:
                        selectedRow.differenceAmount < 0
                          ? "#d32f2f"
                          : "#2e7d32",
                    }}
                  >
                    {selectedRow.differenceAmount?.toLocaleString()}
                  </Typography>
                </Box>
              </Box>
            </Box>
          )}
        </DialogContent>

        <DialogActions sx={{ p: 3 }}>
          <Button
            onClick={() => setOpenHistory(false)}
            variant="contained"
            sx={{
              borderRadius: 3,
              px: 4,
              bgcolor: "#1a237e",
              textTransform: "none",
            }}
          >
            Close
          </Button>
        </DialogActions>
      </Dialog>
    </Container>
  );
};

export default BalanceParticularSearchScreen;
